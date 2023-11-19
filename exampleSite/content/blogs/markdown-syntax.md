---
title: "1er billet de blog"
date: 2023-11-19T23:02:21+01:00
draft: false
github_link: "https://github.com/jseguillon"
author: "jseguillon"
tags:
  - Ansible
  - Débuter
  - Tips and tricks
image: /images/post.jpg
description: ""
toc:
---

Ce premier billet, sur Ansible. Ca fait un bon moment que je pratique cet outil (8 bonnes années je pense) et parfois je vous des porjets qui débutent assez mals. J'écris ce billet dans l'espoir qu'il aide le plus grand nombre à se faciliter la prise en main d'un outil qui a la réputation d'être difficile

## La base pour se rendre la vie plus facile

Plusieurs points peuvent vous aider à avoir une expérience plus douce avec Ansible: 
 - utiliser le check mode,
 - améliorer les logs

(TODO screenshot gauche: une étape, logs affreuses json no color/ droite: deux étapes, logs au top)

### Gloire au check mode 

Vous pouvez simuler l'éxécution d'un playbook Ansible 

### Se soucier des belles logs

l'option -D => affiche les différences induites par un changement,

TODO: screen avt/après. 

#### le default callback:
ANSIBLE_CHECK_MODE_MARKERS pour aller avec le check mode
la variable ANSIBLE_CALLBACK_RESULT_FORMAT=yaml
la variable d'environnement ANSIBLE_SHOW_TASK_PATH_ON_FAILURE=True
la varible ANSIBLE_FORCE_COLOR
ANSIBLE_DISPLAY_OK_HOSTS
ANSIBLE_DISPLAY_SKIPPED_HOSTS

Il y a d'autres callbacks: https://docs.ansible.com/ansible/latest/plugins/callback.html#callback-plugins et aussi Caradoc :) 

NB: vous pouvez aussi régler ces variables via un fichier ansible.ini bien sûr.

#### Flag verbose

le flag -v 
(pensez à le rendre configurable dans les params de la CI ça peut servir)

TODO: command lines
=> check playbook infra
=> deploy infra
=> check par devs
=> deploy par devs

(mais attention au 1er check mode => recette)

## Des inventaires dynamiques

Les inventory plugins https://docs.ansible.com/ansible/2.9/plugins/inventory.html#plugin-list

et rappel: possibles d'en mettre plusieurs sur la ligne de commande 

## Gérer ses dépendances internes

(probablement pas plus d'une collection (sera tjs temps de découper)  )

## gare aux credentials Ansible

(=no_log chiant et même pas secure à cause du ANSIBLE_DEBUG)

## gare à molécule 

(=oui c'est chouette mais est-ce que ça vaut le coup face à la flakyness alors qu'on a le check mode et des pltfs de recette)

