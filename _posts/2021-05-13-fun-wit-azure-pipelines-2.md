---
title:  Fun with Azure Pipelines - if - 2/?
date:   2021-05-13 14:12:15 +0200
tags: ["Azure DevOps", "Azure Pipelines", "DevOps", "Fun with Azure Pipelines"]
categories: fr
thumb: /assets/pipelines.jpg
header:
  teaser: /assets/pipelines.jpg
published: true
---

Cet article fait partie d'une série consacré à Azure Pipelines. Retrouvez l'[épisode précédent](/fr/2021/04/12/fun-wit-azure-pipelines-1.html)

# Au menu aujourd'hui

Aujourd'hui, on va s'intéresser à la meta-tasks ```if```. Il existe des fonctionnalités dans Azure Pipelines qui peuvent remplir ce rôle, mais celles-ci offre des possibilités intéressantes.

Toujours dans le contexte de l'écriture de notre template réutilisable dans n projets, nous allons voir comment le rendre plus souple pour qu'il s'adapte au mieux.

Go !

# Rappel

Un template Azure Pipeline défini des paramètres en input. C'est sur ces paramètres que nous allons nous appuyer pour exploiter ```if``` ainsi que sur les variables système, celles [prédéfinies d'Azure DevOps](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml) et qui contiennent des informations de contexte.

On oublie donc les variables que l'on pourrait définir en cours de pipeline.

# Exécution conditionnelle de stage, job ou step

Il se peut qu'on veuille executer ou non certaines tâches en fonction d'un paramètre particulier. Il existe deux possibilités :

- l'attribut ```condition``` présent sur les éléments stage, job et task
- la méta-task ```if```

## Avec ```condition```

```condition``` existe depuis la génération précédente (pipelines en mode designer), aussi bien côté Build que côté Release.

```yaml
{% raw %}
- job: BuildAllCondition
  steps:
  - powershell: |
      Write-Host "Hello Step 1"
    displayName: Step 1
  - powershell: |
      Write-Host "Hello Step 2"
    condition: ${{ eq(parameters.doEven, true) }}
    displayName: Step 2
  - powershell: |
      Write-Host "Hello Step 3"
    displayName: Step 3
  - powershell: |
      Write-Host "Hello Step 4"
    condition: ${{ eq(parameters.doEven, true) }}
    displayName: Step 4
  - powershell: |
      Write-Host "Hello Step 5"
    displayName: Step 5
  - powershell: |
      Write-Host "Hello Step 6"
    displayName: Step 6
    condition: ${{ eq(parameters.doEven, true) }}
{% endraw %}
```

Le résulat obtenu est similaire à ceci :

![Résultat - Condition](/assets/fun-with-pipe-2/condition-pipe-result.png)

Fonctionnel mais pas des plus esthétiques.

## Avec ```if```

Avec ```if```, le code de notre pipeline devient :

```yaml
{% raw %}
- job: BuildAllIf
  steps:
  - powershell: |
      Write-Host "Hello Step 1"
    displayName: Step 1
  - ${{ if eq(parameters.doEven, true) }}:
    - powershell: |
        Write-Host "Hello Step 2"
      displayName: Step 2
  - powershell: |
      Write-Host "Hello Step 3"
    displayName: Step 3
  - ${{ if eq(parameters.doEven, true) }}:
    - powershell: |
        Write-Host "Hello Step 4"
      displayName: Step 4
  - powershell: |
      Write-Host "Hello Step 5"
    displayName: Step 5
  - ${{ if eq(parameters.doEven, true) }}:
    - powershell: |
        Write-Host "Hello Step 6"
      displayName: Step 6
{% endraw %}
```

Et le résultat obtenu est fonctionnellement identique mais un brun plus propre

![Résultat - if](/assets/fun-with-pipe-2/if-pipe-result.png)

L'intérêt de ```if``` n'est pas qu'esthétique et pour mieux comprendre, d'autres cas d'usage.

# ```if``` et  ```dependsOn ```

Que se passe-t-il si vous voulez rendre une phase facultative en fonction d'un paramètre, le déploiement sur un certain environnement par exemple, mais qu'un stage dépend de celui-ci.

```if``` prend ici tout son sens. Même si la syntaxe est un peu lourde et peut complexifier le pipeline, on gagne énormément en souplesse d'exécution.

```yaml
{% raw %}
parameters:
  - name: build2
    type: boolean
    default: false

stages:
- stage: s1
  dependsOn: []
  displayName: Build
  jobs:
  - template: fakejobs.yml

- ${{ if eq(parameters.build2, true) }}:
  - stage: s2
    dependsOn: []
    displayName: Build
    jobs:
    - template: fakejobs.yml

- stage: d1
  dependsOn:
  - s1
  - ${{ if eq(parameters.build2, true) }}:
    - s2
  displayName: Deploy on Env1
  jobs:
  - template: fakejobs.yml
{% endraw %}
```

Dans cet exemple, on peut choisir d'exécuter le stage s2 en fonction d'une caractéristique d'un projet. Notre pipeline va ainsi s'adapter et gérer proprement l'exécution du déploiement.

Voici le résultat en image :

![Pas de build secondaire](/assets/fun-with-pipe-2/dependson-nobuild2.png)

![Avec build secondaire](/assets/fun-with-pipe-2/dependson-build2.png)

# ```if``` et les variables

Tout comme pour ```dependsOn```, on peut définir des variables en fonction d'un paramètre :

```yaml
{% raw %}
parameters:
  - name: param1
    type: boolean
    default: false

variables:
- ${{ if eq(parameters.param1, true) }}:
  - name: myVar
    value: "One value"
- ${{ if eq(parameters.param1, false) }}:
  - name: myVar
    value: "Another value"

stages:
- stage: s1
  dependsOn: []
  displayName: Build
  jobs:
  - job:
    steps:
    - powershell: |
        Write-Host "Hello $(myVar)"
      displayName: '$(myVar)'
{% endraw %}
```

![Résultat - Variable conditionnelle](/assets/fun-with-pipe-2/conditional-variable.png)

# Conclusion

Le ```if``` offre une option intéressante pour gérer des exécutions conditionnelles. Son évaluation très tôt, avant l'éxécution du pipeline, permet de simplifier la lecture de l'exécution quand appliqué sur des stages, jobs ou steps. Il permet également plus de souplesse pour la gestion des dépendances et éventuellement de variables.

En revanche, il ne remplacera pas l'attribut ```condition``` qui est lui évalué à l'éxécution.

On va donc dire qu'ils sont complémentaires et cet article devrait vous aider à y voir plus clair.