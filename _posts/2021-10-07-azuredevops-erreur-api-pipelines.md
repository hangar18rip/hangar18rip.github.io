---
title: \'No pool was specified\' avec l'API pipelines d'Azure DevOps
date:   2021-10-07 11:40:00 +0200
tags: ["pipelines", "api", "DevOps", "Azure DevOps", "Azure Pipelines", "yaml", "schema"]
categories: fr
thumb: /assets/pipelines.jpg
header:
  teaser: /assets/pipelines.jpg
published: true
---

Quand les choses se répètent, c'est généralement une bonne idée de chercher à automatiser, surtout dans notre domaine. C'est exactement ce que j'étais en train de faire quand je suis tombé sur une erreur 'étrange'.

# Azure DevOps pipeline REST API

Je gère des projets qui se ressemblent. Même stack, même structure, tout ou presque est identique sauf la logique implémentée. Parfait sujet à automatiser. Le projet comprend tout un tas d'éléments commun dont un qui est le pipeline. Chaque projet a son pipeline, basé sur un template, mais il faut quand même le créer, ou l'instancier comme vous voulez, sur chaque projet. Chaque projet est créé à partir d'un template de projet qui contient un 'modèle' de pipeline dans lequel on remplace des placeholders en fonction du projet.

Rien de bien sorcier dans l'absolu, Azure DevOps met à disposition une API REST pour créer un pipeline YAML : [https://docs.microsoft.com/en-us/rest/api/azure/devops/pipelines/pipelines/create?view=azure-devops-rest-6.1](https://docs.microsoft.com/en-us/rest/api/azure/devops/pipelines/pipelines/create?view=azure-devops-rest-6.1)

```json
{
    "folder": "/",
    "name": "nom du pipeline",
    "configuration": {
        "type": "yaml",
        "path": "/.cicd/my-pipeline.yml",
        "repository": {
            "id": "00000000-0000-0000-0000-000000000000",
            "name": "my repo",
            "type": "azureReposGit"
        }
    }
}
```

Pour appeler l'API, je passe par le module PowerShell [VSTeam](https://www.powershellgallery.com/packages/VSTeam/7.4.0) porté par [Donovan Brown](https://github.com/darquewarrior). Il gère l'authentification et pas mal d'autres choses.

L'appel à l'API ressemble donc au snippet suivant :

```powershell
$apiUri = "https://dev.azure.com/myorg/myproject/_apis/pipelines?api-version=6.1-preview.1"
$body = '{
    "folder": "/",
    "name": "my-pipeline",
    "configuration": {
        "type": "yaml",
        "path": "/.cicd/my-pipeline.yml",
        "repository": {
            "id": "00000000-0000-0000-0000-000000000000",
            "name": "my repo",
            "type": "azureReposGit"
        }
    }
}'
Invoke-VSTeamRequest -Url $apiUri -method Post -body $body -Verbose
```

Ce bout de code a fonctionné des dizaines et des dizaines de fois. Jusqu'à hier.

# \'No pool was specified\'

```
WARNING: An error occurred: Response status code does not indicate success: 500 (Internal Server Error).
WARNING: No pool was specified.
```

WTFF ?!

No Pool ... no pool ... pourquoi je devrais spécifier un pool ? Il y a bien un pool par défaut sur les pipelines en YAML et de toutes façons, mon template précise un pool bien précis pour mon besoin. Qu'est-ce que c'est que cette histoire ?

Je bascule en mode recherche et je tombe sur plusieurs articles mentionnant cette erreur :

- [https://stackoverflow.com/questions/68758184/pipeline-creation-with-azure-devops-rest-api-fails-with-error-no-pool-was-speci](https://stackoverflow.com/questions/68758184/pipeline-creation-with-azure-devops-rest-api-fails-with-error-no-pool-was-speci)
- [https://developercommunity.visualstudio.com/t/creating-pipeline-by-api-60-preview1-no-pool-was-s/1327848](https://developercommunity.visualstudio.com/t/creating-pipeline-by-api-60-preview1-no-pool-was-s/1327848)

Je les lis attentivement, regarde mon pipeline dans tous les sens, analyse le template. Rien. Le problème ne vient pas de là.

J'essaie avec Azure cli et le module DevOps. Ca passe mais avec un gros warning à la fin :

```
> az pipelines create --organization "https://dev.azure.com/myorg" --project "myproject" --name 'my-pipeline' --repository "https://ThalesDCC@dev.azure.com/myorg/myproject/_git/myrepo" --branch master --yml-path /.cicd/my-pipeline.yml
This command is in preview and under development. Reference and support levels: https://aka.ms/CLI_refstatus
Successfully created a pipeline with Name: my-pipeline, Id: 0.
Could not queue the build because there were validation errors or warnings.
> _
```

La dernière ligne est intéressante : mon pipeline n'est pas valide. Au passage, pourquoi ça passe avec ```az cli``` et pas l'API, même avec un fichier qui n'existe pas ? A l'occasion, je me pencherai sur ce problème. Mais revenons à notre problème : pourquoi ça ne passe pas avec l'API.

# Comment en est-on arrivé là ?

Puisque mon pipeline est créé avec ```az cli```, essayons de l'exécuter pour voir :

![Mais c'est bien sûr !](/assets/azdo-pipeline-restapi-error/run-error.png)

Ah tiens, le message d'erreur est plus parlant.

L'origine du problème c'est ... moi. Dans un moment d'égarement, j'ai mal déclaré une variable, pure erreur de syntaxe. Mais pas lié pour un cent à un 'No pool was specified'.

# Plus d'excuse !

Les erreurs de syntaxe, ça arrive à tout le monde (note pour plus tard : prendre des pauses et se coucher plus tard). Comment s'en prémunir ?

Pour les pipelines, il existe un schéma JSON maintenu par Microsoft : [https://github.com/microsoft/azure-pipelines-vscode/blob/main/service-schema.json](https://github.com/microsoft/azure-pipelines-vscode/blob/main/service-schema.json)

Avec Visual Studio Code, vous pouvez utiliser l'[extension YAML](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml) de [Red Hat](https://marketplace.visualstudio.com/publishers/redhat). Elle propose deux moyens de valider que votre YAML est valide

Soit vous rajoutez cette ligne au tout début de chaque fichier de Pipeline (bof bof mais ça marche):

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/microsoft/azure-pipelines-vscode/main/service-schema.json
```

Soit vous utilisez une configuration pour vous ou pour le projet :

```
"yaml.schemas": {
  "https://raw.githubusercontent.com/microsoft/azure-pipelines-vscode/main/service-schema.json": "/.cicd/*.yml"
}
```

A vous d'adapter le filtre pour savoir à quels YAML vous voulez appliquer le schéma.

![Pour tester](/assets/azdo-pipeline-restapi-error/validation-test.png)

![Et ça marche !](/assets/azdo-pipeline-restapi-error/validation-error.png)

Une alternative, qui n'est pas valable pour la création d'un pipeline, consiste à utiliser une autre API d'Azure DevOps : [https://docs.microsoft.com/en-us/rest/api/azure/devops/pipelines/preview/preview?view=azure-devops-rest-6.1](https://docs.microsoft.com/en-us/rest/api/azure/devops/pipelines/preview/preview?view=azure-devops-rest-6.1).

Elle suppose que vous avez déjà créé un pipeline et que vous voulez valider une nouvelle version de sa définition. Le module VSTeam comporte une CmdLet pour vous faciliter la vie : [https://methodsandpractices.github.io/vsteam-docs/docs/modules/vsteam/commands/Test-VSTeamYamlPipeline/](https://methodsandpractices.github.io/vsteam-docs/docs/modules/vsteam/commands/Test-VSTeamYamlPipeline/)

Et voilà comment d'un mauvais message d'erreur on améliore nos pratiques.

Have fun with Azure Pipelines ;)