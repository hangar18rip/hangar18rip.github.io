---
layout: post
title:  Fun avec les Azure Pipelines
date:   2021-04-12 14:12:15 +0200
tags: ["Azure DevOps", "Azure Pipelines"]
categories: fr
author: Olivier Delmotte
thumb: /assets/pipelines.jpg
published: false
---

Azure Pipelines, version Yaml, offre des possibilités très intéressantes. Il est vrai que l'absence d'interface d'interface pour éditer les pipelines peut décourager, mais après un peu de pratique, on y prend goût et on se surprend à faire des choses assez intéressantes.

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

# each & if

Les pipeline yaml offrent la possibilité d'utiliser des boucles et des branchement

## each

Boucler avec le précedent système de pipeline n'était pas possible.


## if

