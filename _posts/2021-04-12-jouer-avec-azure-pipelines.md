---
title:  Fun avec les Azure Pipelines
date:   2021-04-12 14:12:15 +0200
tags: ["Azure DevOps", "Azure Pipelines"]
categories: fr
thumb: /assets/pipelines.jpg
published: false
---

Je travail actuellement sur des projets de même nature pour un client. Chaque projet vit sa vie pour les besoins business mais leur architecture de base est la même.

Pour simplifier la maintenance de ces projets, nous avons décidé de mutualiser au maximum : practices, scripts IaC, pipelines.

Si pour les practices et les scripts IaC la chose est plus habituelle, c'était un peu moins l'habitude sur les pipelines depuis l'abandon du système basé sur Workflow Foundation. Excepté les Task Groups, il était difficile de faire du reuse cross projet.

C'est une grosse occasion de tester les limites du système de template Azure Pipelines version Yaml.

Ce système offre des possibilités très intéressantes. Il est vrai que l'absence d'interface d'interface pour éditer les pipelines peut décourager, mais après un peu de pratique, on y prend goût et on se surprend à faire des choses assez intéressantes.

# Pipeline as Code

Comme la définition du Pipeline est définie avec le code, si le développement du produit implique une modification du pipeline, on peut le faire sur la branche en cours. Le développement des évolutions peut donc se faire sans impacter les autres branches en cours de développement.

# Templating

Première chose intéressante avec Azure Pipelines, c'est la possibilité de définir des templates. Il est possible de définir des templates pour les 3 niveaux d'exécution du Pipeline :
- un template de pipeline entier
- un template de phase
- un template de job

## Templating de pipeline entier

Dans certains cas, il est possible de définir un template pour un pipeline entier. Quand on doit gérer plusieurs projets avec la même structure, écrire un pipeline pour chacun peut se révéler fastidieux. Avec les templates, les copier / coller sont réduits au maximum.

```yaml
name: 0.$(date:yyyy).$(date:Mdd).$(rev:r)

trigger:
  branches:
    include:
    - '*'

resources:
  repositories:
  - repository: pipelines
    type: git
    name: templates/azure-pipelines

extends:
  template: sample.yml@pipelines
  parameters:
      param1: value1
      param2: value2
      param3: value3
```

## Templating de stages / jobs / steps

Dans ce cas là, on facilite la lecture et le développement des pipelines. On peut aussi réutiliser des phases ou des jobs dans d'autres pipelines. Bref, on facilite le partage.

Ce genre de possibilité permet de factoriser certaines parties de pipelines : déploiement Infra as Code, exécution de tests, etc. Toujours dans l'optique mutualiser les efforts.

# ```each``` & ```if```

Les pipeline yaml offrent la possibilité d'utiliser des boucles et des branchement

## ```each```

Boucler avec le précedent système de pipeline n'était pas possible (si on fait abstraction des matrices).

Dans Azure Pipelines, on peut toujours utiliser les matrices pour compiler un même code pour plusieurs plate-formes, mais on peut aussi boucler à loisir :
- au niveau stage, pour boucler sur une liste d'environnement par exemple. On en parle plus loin
- au niveau jobs aussi et tâches
- sur un ou plusieurs éléments regroupés

Bref, très grosse souplesse d'action.

## ```if```

Comme pour les boucles, même souplesse. Il existe déjà l'attribut ```condition``` sur les éléments ```stage```, ```job``` & ```task```. Mais condition implique de définir l'attribut sur chaque élémént. ```if``` permet de regrouper plusieurs éléments.

Autre avantage, si ```if``` est évalué à ```false```, l'élément n'apparaît pas dans le log du run.

Rien n'empêche de combiner les deux systèmes.

# Advanced templating

Quand on doit faire du reuse sur plusieurs projets, on fait face à plusieurs contraintes :
- 1 modification impacte n projets
- certains projets ont quand même des particularités : composants supplémentaires, composants activés ou non
- le nombre d'environnements
- une ou plusieurs cibles de déploiement (ex : cloud + onprem)
- ...

Il faut donc pouvoir adapter le template à ces particularités

