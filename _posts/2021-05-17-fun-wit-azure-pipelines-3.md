---
title:  Fun with Azure Pipelines - each - 3/?
date:   2021-05-17 14:13:15 +0200
tags: ["Azure DevOps", "Azure Pipelines", "DevOps", "Fun with Azure Pipelines"]
categories: fr
thumb: /assets/pipelines.jpg
published: true
---

Cet article fait partie d'une série consacré à Azure Pipelines. Retrouvez les épisodes précédents :

- [Episode 1 - Intro](/fr/2021/04/12/fun-wit-azure-pipelines-1.html)
- [Episode 2 - if](/fr/2021/05/13/fun-wit-azure-pipelines-2.html)

# Au programme

On a vu, dans l'article précédent, la méta-task ```if```. Dans celui-ci, nous allons voir la méta-task ```each``` et il sera bien plus court. Déjà, parce qu'il n'y a pas d'équivalent dans la version "UI" des pipelines Azure Pipelines et que la comparaison avec la fonctionnalité [Matrix](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema%2Cparameter-schema#matrix) ne tient pas la route.

Passons aux choses sérieuses et commençons à jouer avec ```each```

# ```each``` - level Padawan

Comme son nom l'indique, ```each``` va nous permettre de réaliser une boucle. Oui mais dans quelles conditions.

Comme son cousin ```if```, ```each``` va fonctionner avec des paramètres. Du coup, pas de ```each``` en fonction d'une variable, définie dans le template ou récupérée d'une sortie de ```task```. ```each``` va fonctionner sur des paramètres de type ```object```.

Dans ce premier scénario, on va utiliser un simple tableau d'item. Le cas d'usage typique que j'en ai dans mes template : quand je dois itérer sur une liste de fichiers particuliers pour exécuter des tâches particulières.

```yaml
{% raw %}
parameters:
- name: simpleIterator
  type: object
  default:
  - item1
  - item2
  - item3

stages:
- stage: s1
  dependsOn: []
  displayName: Build
  jobs:
  - job:
    steps:
    - ${{ each value in parameters.simpleIterator }}:
      - powershell: |
          Write-Host "Hello ${{ value }}"
        displayName: '${{ value }}'
{% endraw %}
```

Si je queue mon pipeline en rajoutant une entrée à ma liste de valeur :
![Loop level 1](/assets/fun-with-pipe-3/loop-level1-queue-param.png)

On retrouve bien mes 4 tasks PowerShell avec le nom de chaque entrée de mon paramètre ```simpleIterator``` et la sortie associée.

![Loop level 1](/assets/fun-with-pipe-3/loop-level1.png)

Efficace non ? Mais on peut aller encore plus loin avec ```each```

# ```each``` - level Chevalier

C'est le cas d'usage que j'ai trouvé le plus intéressant dans Azure Pipelines, comparé à d'autres système que j'ai pu côtoyer.

J'utilise ce scénario pour définir, par configuration, des environnements. Chaque projet étant unique même si ils peuvent partager des éléments, ```each``` m'apporte une énorme souplesse.

Avec cette définition de pipeline :

```yaml
{% raw %}
parameters:
- name: myEnvironments
  type: object
  default:
      env1:
      param1: 'v1'
      dependsOn: []
      env2:
      param1: 'v2'
      dependsOn: ['env1']

stages:
- ${{ each myEnvironment in parameters.myEnvironments }}:
  - stage: '${{ myEnvironment.Key }}'
    dependsOn: ${{ myEnvironment.Value.dependsOn }}
    displayName: 'Deploying on ${{ myEnvironment.Key }}'
    jobs:
    - job: '${{ myEnvironment.Key }}'
      steps:
      - powershell: |
          Write-Host "Hello ${{ myEnvironment.Value.param1 }}"
        displayName: '${{ myEnvironment.Value.param1 }}'
{% endraw %}
```

J'obtiens ce résultat :

![Loop level 2](/assets/fun-with-pipe-3/loop-level2-2envs.png)

En modifiant mon paramètre définissant mes environnements de la sorte :

```yaml
{% raw %}
parameters:
- name: myEnvironments
  type: object
  default:
    env1:
      param1: 'v1'
      dependsOn: []
    env2:
      param1: 'v2'
      dependsOn: ['env1']
    env3:
      param1: 'v3'
      dependsOn: ['env1']
    env4:
      param1: 'v4'
      dependsOn: ['env1']
    env5:
      param1: 'v5'
      dependsOn: ['env2', 'env3', 'env4']
    env6:
      param1: 'v5'
      dependsOn: ['env5']
{% endraw %}
```

J'obtiens :

![Loop level 2](/assets/fun-with-pipe-3/loop-level2-6envs.png)

Et comme on manipule des objets, on peut avoir quelque chose d'assez poussé et performant.

# ```each``` - level Maître

Et si on imbriquait 2 ```each``` ? Allez !

Très utile dans des scénarios où vous devez adapter l'exécution à un environnement. Par exemple, un script à jouer après le déploiement sur un environnement pour restaurer une base de données en vue de lancer des tests, ou justement que vous voulez jouer une famille de tests sur un environnement. Plein de scénarios possibles, mais c'est ce dernier que je vais retenir pour l'exemple suivant :

```yaml
{% raw %}
parameters:
- name: myEnvironments
  type: object
  default:
    env1:
      dependsOn: []
      iac:
        variableGroup: 'varGroup1'
      app:
        param1: 'value1.1'
        param2: 'value1.2'
    env2:
      dependsOn: ['env1']
      iac:
        variableGroup: 'varGroup2'
      app:
        param1: 'value2.1'
        param2: 'value2.2'
      test:
      - test1
      - test2


stages:
- ${{ each myEnvironment in parameters.myEnvironments }}:
  - stage: '${{ myEnvironment.Key }}'
    dependsOn: ${{ myEnvironment.Value.dependsOn }}
    displayName: 'Deploying on ${{ myEnvironment.Key }}'
    jobs:
    - job: 'iac${{ myEnvironment.Key }}'
      steps:
      - powershell: |
          Write-Host "Deploying infra on Azure using variable group ${{ myEnvironment.Value.iac.variableGroup }}"
        displayName: 'Deploying infra'
    - job: 'deployApp${{ myEnvironment.Key }}'
      steps:
      - powershell: |
          Write-Host "Deploying app on Azure - ${{ myEnvironment.Value.app.param1 }}"
        displayName: 'Deploying app'
    - ${{ if ne(coalesce(myEnvironment.Value.test, ''), '') }}:
      - job: 'testApp${{ myEnvironment.Key }}'
        steps:
        - ${{ each test in myEnvironment.Value.test }}:
          - powershell: |
              Write-Host "Testing on Azure - test : ${{ test }}"
            displayName: 'Testing app ${{ test }}'
{% endraw %}
```

Le résultat est le suivant :

![Loop level 3](/assets/fun-with-pipe-3/loop-level3.png)

Et on voit bien que j'ai bouclé sur chaque test défini dans mon paramètre :

![Loop level 3](/assets/fun-with-pipe-3/loop-level3-details.png)

# Conclusion

Avec son compère ```if```, ```each``` nous redonne un peu de la puissance que l'on a perdu en passant du système de Build introduit avec TFS 2010 basé sur Workflow Foundation mais pas encore totalement. Il faut se souvenir qu'on est limité au bouclage sur des paramètres et non pas sur des variables. Pour rappel, les paramètres sont définis en lecture seule et évalués avant l'exécution du Pipeline.

Néanmoins, on gagne en souplesse pour définir et réutiliser des templates sur plusieurs projets.
