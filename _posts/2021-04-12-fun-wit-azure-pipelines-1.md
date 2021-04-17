---
layout: post
title:  Fun with Azure Pipelines - Intro - 1 of ...
date:   2021-04-12 14:12:15 +0200
tags: ["Azure DevOps", "Azure Pipelines", "DevOps"]
categories: fr
author: Olivier Delmotte
thumb: /assets/pipelines.jpg
published: false
---

Chez un de mes clients, je suis en charge, entre autre, de l'usine de livraison, basée sur Azure DevOps et donc Azure Pipelines. Dans ce cadre, j'accompagne une multitude d'équipes sur des projets très similaires niveau architecture et donc, similaires dans leur façon d'être construits et déployés.

C'est l'occasion parfaite de pousser un peu plus loin mes expérimentations sur Azure Pipelines, version Yaml, et de jouer avec certains concepts.

# Projet #1 - la v0.0

Pas besoin de rentrer dans les détails sur la stack technique des projets, mais plutôt sur quelques principe et un peu d'historique.

A la base, mon intervention était très ponctuelle, pour dépanner une équipe en retard sur l'implémentation de la chaîne CI / CD et la transcription de l'architecture en scripts Terraform.

J'ai donc passé un bon mois a développer, avec l'équipe, ce pipeline de déploiement et les modules Terraform.

![Run du Pipeline](/assets/simple-yaml-pipeline.png)

Devant l'urgence et le scope, l'ensemble a été intégré au projet. Même si il y avait un début de découpage du pipeline en templates pour faciliter la lecture, celui-ci restait très lié au projet : les noms de projets étaient en dur dans le yaml, le nom du projet se retrouvait aussi un peu partout.

Ca faisait le job pour répondre à l'urgence et rattraper le retard.

Mais devant la réussite et l'efficacité du résultat, ça a donné des idées au client pour appliquer tout ça à d'autres projets du même type. Damn !

# Projet #2 - la v0.1

On m'a retrouvé un projet qui était lui aussi un peu (beaucoup) en retard avec une contrainte de temps toujours aussi serrée.

La décision a été prise de dupliquer les pipelines et de se focaliser sur la mutualisation du code de l'infra dans un premier temps, histoire d'optimiser [mon flow](https://www.scrum.org/PSK).

L'optimisation / mutualisation des pipelines se ferait dans un deuxième temps.

# Et voilà la v1

Cette solution de copier / collé a plutôt bien fonctionné pour les pipelines. Et avec les pipelines des deux projets gérés par la même personne, il était assez facile de reporter les updates / fixes entre les deux version. Mais ça ne peut pas durer ! Que va-t-il se passer quand il y aura 10 projets similaires en parallèle ?

Il était temps de mettre à profit la petite accalmie pour refactorer un peu tout ça

# Azure Pipelines : les templates

Même si le pipeline de base et sa copie étaient déjà découpés, il fallait faire mieux :
- Sortir les définitions de pipelines des projets
- S'assurer que les templates n'étaient pas spécifiques à un projet
- Assurer la transition entre le mode tout intégré et le mode templatisé

Voici à quoi représente la définition du pipeline, dans une version simplifiée, avant :

```yaml
trigger:
- main

pool:
  vmImage: ubuntu-latest

stages:
- stage: Build
  displayName: Build
  jobs:
  - template: build/build-jobs-template.yml
- stage: DeployOnEnv1
  displayName: Deploy on Env1
  dependsOn: Build
  jobs:
  - template: deploy/deploy-jobs-template.yml
- stage: DeployOnEnv2
  displayName: Deploy on Env2
  dependsOn: DeployOnEnv1
  jobs:
  - template: deploy/deploy-jobs-template.yml
- stage: DeployOnEnv3
  displayName: Deploy on Env3
  dependsOn: DeployOnEnv2
  jobs:
  - template: deploy/deploy-jobs-template.yml
- stage: DeployOnEnvA
  displayName: Deploy on EnvA
  dependsOn: Build
  jobs:
  - template: deploy/deploy-jobs-template.yml
```

## Externalisation du template

La première chose à faire était de sortir une copie du template. Oui, une copie.

La transition va prendre un certain temps et le projet doit continuer à travailler.

Pour marquer cette externalisation, nous avons créé un nouveau projet transverse, qui héberge les briques partagées : templates de pipeline, modules terraform, ;..

La copie rejoint donc un repository dédié dans ce nouveau projet et qui contiendra ce template plus tous les autres à suivre.

Comme tout bon développeur qui se respecte, nous allons tirer une branche sur nos projets pour réaliser la migration et câbler le pipeline vers le template et y référencer notre template de pipeline.

Sur notre branche de migration, le code de notre pipeline devient donc :
```yaml
trigger:
- main

pool:
  vmImage: ubuntu-latest

resources:
  repositories:
  - repository: pipelines
    type: git
    name: AzureDevOps/azure-pipelines
    ref: refs/heads/main

extends:
  template: base.yml@pipelines
```

On ajoute une référence au repository qui contiendra notre template de pipeline via un [Repository Resource](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/resources?view=azure-devops&tabs=schema#resources-repositories).

Deux points intéressants sur cette référence :
- On peut ajouter une référence vers un repository git dans [Azure DevOps](https://dev.azure.com), un repo [GitHub](https://github.com/) et même [BitBucket](https://bitbucket.org/product/).
- On peut spécifier une version particulière via ```ref``` vers une branche, un tag ou un commit particulier. On y reviendra plus tard dans le "lesson learned". En phase dev, on reste sur la ```main```, on gagnera du temps.

On peut d'ailleurs voir que 2 repositories sont présents dans l'output du Pipeline :
![Sources du Pipeline](/assets/template-pipe-sources.png)

La première version de ce template sera typée pour un projet et pas très fonctionnelle mais pas pour longtemps.

## Anonymisation du template

La deuxième étape consiste à anonymiser ce projet :
- identifier tous les éléments "hard-codés" correspondant au premier projet
- les remplacer par des paramètres
- tester !
- itérer

## Transition

La transition devrait se faire en douceur.

# Versioning

Versionner les templates est une grosse problématique. Voici la solution qui me convient, dans ce scénario et d'autres, mais qui peut ne pas vous convenir.

Les versions **majeures** de production sont identifiées par un tag posé sur la branche ```master``` ou ```main```. Dans mon cas, il prend la forme ```v1.0.0``` avec un message dont on ne tient pas compte.

Cela permet de référencer le template de la façon suivante :
```yaml
# ...

resources:
  repositories:
  - repository: pipelines
    type: git
    name: AzureDevOps/azure-pipelines
    ref: refs/tags/v1.0.0

# ...
```

Une nouvelle version majeure est créée lorsqu'un breaking change est introduit : nouveau paramètre mandatory, comportement altéré de façon majeure.

Pour les version mineurs (small feature change ou bug fix), le tag de la version en cours est recréé sur la version la plus à jour.

L'attribut ref de l'élément repository est intéressant dans la mesure où il permet d'exécuter un pipeline sur une version spécifique :
- version de développement sur une branche particulière
- version en preview identifiée par un tag par exemple (```v2.0.0-beta```)
- de laisser le temps aux projets de migrer d'une version majeure à une autre.

Le tag permet par ailleurs de faire un rollback rapidement en cas de bug majeur sur un template.

# Résultat final

## Côté templates

Dans le repo des templates :


Le template ```base.yaml```

```yaml
# pipeline template
parameters:
- name: projectName
  type: string
  default: Project Z

stages:
- stage: Build
  displayName: Build ${{ parameters.projectName }}
  jobs:
  - template: build/build-jobs-template.yml
    parameters:
      projectName: ${{ parameters.projectName }}
- stage: DeployOnEnv1
  displayName: Deploy on Env1 on ${{ parameters.projectName }}
  dependsOn: Build
  jobs:
  - template: deploy/deploy-jobs-template.yml
    parameters:
      projectName: ${{ parameters.projectName }}
- stage: DeployOnEnv2
  displayName: Deploy on Env2 on ${{ parameters.projectName }}
  dependsOn: DeployOnEnv1
  jobs:
  - template: deploy/deploy-jobs-template.yml
    parameters:
      projectName: ${{ parameters.projectName }}
- stage: DeployOnEnv3
  displayName: Deploy on Env3 on ${{ parameters.projectName }}
  dependsOn: DeployOnEnv2
  jobs:
  - template: deploy/deploy-jobs-template.yml
    parameters:
      projectName: ${{ parameters.projectName }}
- stage: DeployOnEnvA
  displayName: Deploy on EnvA on ${{ parameters.projectName }}
  dependsOn: Build
  jobs:
  - template: deploy/deploy-jobs-template.yml
    parameters:
      projectName: ${{ parameters.projectName }}
```

Le template ```build/build-jobs-template.yml```

```yaml
parameters:
- name: projectName
  type: string

jobs:
- job: BuildPart1
  displayName: Build Part 1 of ${{ parameters.projectName }}
  steps:
  - powershell: |
      Write-Host "Hello from the build part 1 job"
- job: BuildPart2
  displayName: Build Part 2 of ${{ parameters.projectName }}
  steps:
  - powershell: |
      Write-Host "Hello from the build part 2 job"
```

Le template ```deploy/deploy-jobs-template.yml```

```yaml
parameters:
- name: projectName
  type: string

jobs:
- job: DeployInfra
  displayName: Deploy Infra on ${{ parameters.projectName }}
  steps:
  - powershell: |
      Write-Host "Deploying infra on ${{ parameters.projectName }}"
- job: DeployApp
  displayName: Deploy App on ${{ parameters.projectName }}
  steps:
  - powershell: |
      Write-Host "Deploying App on ${{ parameters.projectName }}"
```

## Côté projet

```yaml
trigger:
- main

pool:
  vmImage: ubuntu-latest

resources:
  repositories:
  - repository: pipelines
    type: git
    name: AzureDevOps/azure-pipelines
    ref: refs/heads/main

extends:
  template: base.yml@pipelines
  parameters:
    projectName: MyProj
```

# Takeaway

## Lessons learned

- Si j'ai bien pensé à découper mon template dès le début pour simplifier la lecture et faciliter la réutilisation de jobs, j'aurais du partir à ce moment là sur des templates de pipeline
- Il faut développer au maximum le principe de convention
- Il faut, en combinaison avec le principe de convention, un maximum de valeurs par défaut
- Si on centralise les templates de pipeline à un endroit, on augment aussi d'une certaine manière le risque d'erreur. Donc, comme pour une application normale : **tester**, **tester**, **tester**


## Visual Studio Code

Je vous recommande vivement de travailler depuis [Visual Studio Code](https://code.visualstudio.com/) et d'installer l'extension [Azure Pipelines](https://marketplace.visualstudio.com/items?itemName=ms-azure-devops.azure-pipelines)

Dans le dossier .vscode de votre repository de templates, ajoutez le fichier ```extensions.json``` avec le contenu suivant :
```json
{
    "recommendations": [
        "ms-azure-devops.azure-pipelines"
    ]
}
```
et ce le paramètre suivant dans le fichier ```settings.json``` :
```json
"files.associations": {
    "**/*.yml": "azure-pipelines"
}
```

exemple de  ```settings.json```:
```json
{
    "workbench.colorCustomizations": {
        "activityBar.activeBackground": "#3399ff",
        "activityBar.activeBorder": "#bf0060",
        "activityBar.background": "#3399ff",
        "activityBar.foreground": "#15202b",
        "activityBar.inactiveForeground": "#15202b99",
        "activityBarBadge.background": "#bf0060",
        "activityBarBadge.foreground": "#e7e7e7",
        "statusBar.background": "#007fff",
        "statusBar.foreground": "#e7e7e7",
        "statusBarItem.hoverBackground": "#3399ff",
        "titleBar.activeBackground": "#007fff",
        "titleBar.activeForeground": "#e7e7e7",
        "titleBar.inactiveBackground": "#007fff99",
        "titleBar.inactiveForeground": "#e7e7e799"
    },
    "peacock.color": "#007fff",
    "files.associations": {
        "**/*.yml": "azure-pipelines"
    }
}
```

# Note

Si je comprends les motivations de remplacer ```master``` par ```main```, ça rend l'écriture de doc / article / pipelines plus compliquée !!!