---
layout: post
title:  "Gérer plusieurs personnalités dans git"
date:   2021-04-11 14:12:15 +0200
tags: git tips consulting
categories: blog #git tips consulting
author: Olivier Delmotte
thumb: /assets/rtfm.jpg
language: fr
# permalink: /blog/multiple-identity-git
---
En tant que consultant, je travaille régulièrement pour des clients différents en plus des projets internes et même quelques projets persos sur GitHub ou autre.

Jusque-là, rien d’extraordinaire. Mais c’est vrai qu’en plusieurs années, j’ai souvent utilisé la même adresse e-mail pour mes commits. En soit, rien de bloquant. La plupart des systèmes ne sont pas configurés pour valider l’adresse e-mail associée au commit, l’authentification se faisant lors du Push.

# Quelles solutions ?

Dans git, il existe 3 niveaux de configuration : local, global, system.

Par défaut, quand on définit les valeurs de configuration user.name et user.email, c’est au niveau global. L’information est enregistrée dans le .gitconfig à la racine du profil utilisateur.

La configuration au niveau system impacte tous les utilisateurs de la machine. Je suis seul sur la mienne, mais ça ne change rien au problème.

La première solution serait de modifier au niveau local, c’est-à-dire dans chaque repository.

Avec un petit script maison, c’est jouable. Si les vos sources sont rangées de façon structurées, c’est même plus que jouable.

Mais une autre solution existe et elle me semble beaucoup plus propre et efficace.
# Illustration
En supposant qu’on utilise cette structure de dossier pour classer nos sources :

![Notre arborescence de repos](/assets/arbo.png)

Chaque dossier projet est un repo Git. Si on regarde le l’adresse e-mail associé à chaque repo :
```powershell
gci -Directory -Filter '.git' -Force -Recurse | % { push-location $_ ; $email = git config user.email ; Pop-Location; New-Object psobject -Property @{ path = $_.FullName; email = $email }}
```
Voici le résultat :
```
path                     email
------------------------ ------------
\src\client1\projet\.git mon@mail.com
\src\client2\projet\.git mon@mail.com
\src\client3\projet\.git mon@mail.com
\src\perso\projet\.git   mon@mail.com
\src\pro\projet\.git     mon@mail.com
```
Le résultat est celui auquel on pouvait s’attendre. Mais pas terrible. Si je pousse du code pour un projet du client 1 avec cette configuration, mes commits seront identifiés par le mail courant.
# Implémentation de la solution 1
Implémentons la solution 1 : un script qui met à jour la variable user.email de chaque repo avec une valeur adaptée à chaque situation.

Cette commande PowerShell, exécutée depuis mon dossier src/pro par exemple, me permet de mettre mon adresse pro sur chaque repo contenu dans l’arborescence :
```powershell
gci -Directory -Filter '.git' -Force -Recurse | % { push-location $_ ; git config --local user.email "mail@pro.com" ; Pop-Location;}
```
Si je relance la commande de listing dans mon dossier src\, voici maintenant le résultat :
```
path                     email
------------------------ ------------
\src\client1\projet\.git mon@mail.com
\src\client2\projet\.git mon@mail.com
\src\client3\projet\.git mon@mail.com
\src\perso\projet\.git   mon@mail.com
\src\pro\projet\.git     mail@pro.com
```
Cool. Mais que se passe-t-il si je rajoute un autre repo pour projet2 ?
```
path                     email
------------------------ ------------
\src\client1\projet\.git mon@mail.com
\src\client2\projet\.git mon@mail.com
\src\client3\projet\.git mon@mail.com
\src\perso\projet\.git   mon@mail.com
\src\pro\projet\.git     mail@pro.com
\src\pro\projet2\.git    mon@mail.com
```
Pas terrible comme résultat. On repasse sur le mail par défaut.
# Solution 2
Il existe une autre solution, plus simple et plus efficace. Non pas que ma petite ligne de script PowerShell soit compliquée, mais il faut penser à l’exécuter à chaque nouveau repo.

Non, la solution se cache dans la configuration de git elle-même :

Il existe dans git config depuis quelques versions une fonctionnalité documentée sur la page liée. Cette fonctionnalité est :

```
includeIf
```

Elle permet d’utiliser une configuration particulière en fonction du dossier depuis lequel la commande git est exécutée (mais pas que, lire la doc). Il suffit de la combiner avec ```gitdir/i``` filtrer par dossier. ```/i``` est particulièrement intéressante sous Windows dont le système de fichier n’est pas sensible à la casse. Sous Linux, il est moins critique même s’il reste utilisable.

Dans le cadre de cet exemple, mon fichier de configuration git, le fichier .gitconfig situé à la racine du profil utilisateur, ressemble à ça :

```ini
[user]
    name = Olivier Delmotte
    email = mon@mail.com
[includeIf "gitdir/i:c:/src/perso/"]
    path = /src/.gitconfigperso
[includeIf "gitdir/i:c:/src/pro/"]
    path = /src/.gitconfigpro
[includeIf "gitdir/i:c:/src/client1/"]
    path = /src/.gitconfigclient1
[includeIf "gitdir/i:c:/src/client2/"]
    path = /src/.gitconfigclient2
[includeIf "gitdir/i:c:/src/client3/"]
    path = /src/.gitconfigclient3
```

Je redéfinis un fichier de config pour chaque paramètre que je souhaite surcharger. Dans cet exemple, je spécifie un nouveau mail pour chaque cas. Voici un exemple de fichier pour mon repo pro, situé ici c:\src\pro\.gitconfigpro :

```ini
[user]
     name = Olivier Delmotte (Pro)
     email = mail@pro.com
```

Il suffit de répéter l’opération pour chaque cas.

Au final, si je relance ma commande magique pour lister le mail de chaque repo :

```
path                     email
------------------------ -------------------------------------
\src\client1\projet\.git olivier.delmotte@external.client1.com
\src\client2\projet\.git olivier.delmotte@external.client2.com
\src\client3\projet\.git olivier.delmotte@external.client3.com
\src\perso\projet\.git   mail@perso.fr
\src\pro\projet\.git     mail@pro.com
\src\pro\projet2\.git    mail@pro.com
```

Déjà, pas obligé de lancer mon script pour mettre à jour les autres catégories de repos. Mais en plus, si je rajoute un nouveau projet pour client1 par exemple :

```
path                      email
------------------------- -------------------------------------
\src\client1\projet\.git  olivier.delmotte@external.client1.com
\src\client1\projet2\.git olivier.delmotte@external.client1.com
\src\client2\projet\.git  olivier.delmotte@external.client2.com
\src\client3\projet\.git  olivier.delmotte@external.client3.com
\src\perso\projet\.git    mail@perso.fr
\src\pro\projet\.git      mail@pro.com
\src\pro\projet2\.git     mail@pro.com
```

Il reste encore un défaut : ça ne prend pas en compte un nouveau client tout seul.

# Ce qu’il faut retenir

![RTFM !](/assets/rtfm.jpg)

RTFM qu’ils disaient ! Ben oui, la doc de git config en parle assez clairement.

Cette feature de git config offre pas mal d’autres possibilités. On peut également avoir une configuration particulière en fonction de la branche sur laquelle on se trouve.