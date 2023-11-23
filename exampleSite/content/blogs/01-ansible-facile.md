---
title: "Une meilleure expérience  Ansible"
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

Ansible a mauvaise réputation en terme d'éxpérience, que ce soit pour l'ops/sre qui peine parfois à produire ses YAMLs, ou pour le dev qui voudrait au moins pouvoir lancer quelques pipelines lui-même mais qui ne comprend rien aux logs et qui a peur de faire une bêtise. Ce billet de blog, tout premier pour moi, est là pour donner quelques pistes pour se rendre la vie plus belle quand on utilise Ansible. Let's go!

## L'option diff 

On commence par une option indispensable de la commande `ansible-playbook`: l'option `-D`, diff. Cette option va vous permettre de visualiser les modifications effectuées par une tâche Ansible. Une démo valant mieux que 100 mots, voici un avant/après option `-D`.

L'exemple ci-dessous montre la commande `ansible-playbook -D` en action: 

![Le diff mode en action](/images/blogs/01-ansible-facile/diff_mod.png "Le diff mode en action")

Vous l'aurez compris, l'option nous permet de voir au-délà du simple statut `[changed]`, les modifications **rééellement apportées** par Ansible. Si vous voulez mon avis, cette option devrait être activée par défaut dans Ansible...

La plupart des modules fonctionnent proprement avec le diff mode mais faites la chasse aux tâches qui affichent du changed sans aucune diff. J'ai un souvenir douloureux d'un module qui, renvoyait simplement changed sans détails. Tout le monde s'en était accomodé, les changements étaient maîtrisés sur ce périmètre... jusqu'au jour où...

Néanmoins: toujours lancer `ansible-playbook -D`, toujours!

## L'option check

Autre option indispensable: le flag `-C `, check mode. Si Ansible ne peut garantir l'exécution d'un playbook, comme le ferait un Terraform, il est tout de même possible de lancer Ansible pour qu'il nous affiche ce *qu'il aurait fait* s'il n'avait pas été lancé avec le flag `-C`. 

Dans l'exemple ci-dessous, je lance deux fois le playbook, le fichier n'est jamais modifié:
![Le check mode en action](/images/blogs/01-ansible-facile/check_mod.png "Le check mode en action")

Combiné avec le diff mode `-D` c'est mon combo préféré:
* c'est rassurant de pouvoir vérifier ce qu'il devrait se passer quand on va réellement appliquer avant de le faire effectivement; et indispensable si vous espérez qu'un jour vos devs cliquent d'eux-même sur les étapes de déploiements des pipelines applicatives,
* pour travailler ses roles c'est très pratique: on peut repartir de zéro à chaque exécution, sans jamais devoir re-créer sa VM (ou serveur) avant de recommencer. Bien sûr on doit ensuite s'assurer que tout fonctionne sans le check mode, mais on peut vraiment avancer plus rapidement. Essayez, vous verrez!

**Attention**: cela ne *garanti pas* ce qu'il se passera réellement sur vos machines *au moment* ou vous allez lancer le playbook sans flag `-C`. Entre le moment où vous lancer votre check et le moment ou vous appliquerez réellement votre playbook, il peut y avoir eu des changements sur vos plateformes. Et contrairement à Terraform qui bloquerait en constatant que l'état de départ ne correspond plus à son "plan"; Ansible lui déroulera le playbook jusqu'au bout sans sourciller.

**Attention 2**: les roles ne sont pas souvent compatible avec le check mode, surtout à la première installation. Attendez-vous à devoir ajouter quelques `when: not ansible_check_mode` (voir même des `ignore_errors: {{ ansible_check_mode }}`) dans le code source des rôles que vous forkez.

## Le "default callback" et ses configurations:
TODO: compléter

Quand vous lancer vos playbooks, c'est le "Default callback" qui se charge d'afficher les résultats. Et la configuration par défaut peut s'améliorer en réglant plusieurs variables d'environnement.

* la variable `ANSIBLE_CHECK_MODE_MARKERS` pour aller avec le check mode: permet de bien indiquer si des tâches ont été lancées en check mode ou pas
* la variable `ANSIBLE_CALLBACK_RESULT_FORMAT=yaml`: indispensable pour la lisibilité des 
* la variable `ANSIBLE_SHOW_TASK_PATH_ON_FAILURE=True`

![Yaml](/images/blogs/01-ansible-facile/yaml_format.png "Yaml")

et encore: 
* la variable `ANSIBLE_DISPLAY_OK_HOSTS=True` 
* la variable `ANSIBLE_DISPLAY_SKIPPED_HOSTS=true`

TODO: command lines tableau dev/vs infra
=> check playbook infra
=> deploy infra
=> check par devs
=> deploy par devs

Une dernière pour la route : la variable `ANSIBLE_FORCE_COLOR=True`

TODO ajouter ref callback default plugin

## Utiliser/ajouter un autre callback
TODO

Il y a d'autres callbacks: https://docs.ansible.com/ansible/latest/plugins/callback.html#callback-plugins et aussi Caradoc :) 

NB: vous pouvez aussi régler ces variables via un fichier ansible.ini bien sûr.



## Flag verbose

le flag -v 
(pensez à le rendre configurable dans les params de la CI ça peut servir)


(mais attention au 1er check mode => recette)

## Utiliser les LLMS et les outils IDE 
TODO: 

<!--

## Consulter les magics variables

play_hosts, groups, etc... 


## Des inventaires dynamiques

Les inventory plugins https://docs.ansible.com/ansible/2.9/plugins/inventory.html#plugin-list

et rappel: possibles d'en mettre plusieurs sur la ligne de commande 

## Gérer ses dépendances internes

(probablement pas plus d'une collection (sera tjs temps de découper)  )

## gare aux credentials Ansible

(=no_log chiant et même pas secure à cause du ANSIBLE_DEBUG)

## gare à molécule 

(=oui c'est chouette mais est-ce que ça vaut le coup face à la flakyness alors qu'on a le check mode et des pltfs de recette)

## Produire moins de Ansible ?






## Pour plus tard: 
delegates localhost et connexion local + multiple local

Faire des diffs avec set_facts_diff 

play_host 

forkez vos roles -->