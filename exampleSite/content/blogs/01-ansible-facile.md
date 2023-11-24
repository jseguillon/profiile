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

```
# Toujours mettre le flag -D
ansible-playbook -D ...
```

La plupart des modules fonctionnent proprement avec le diff mode mais faites la chasse aux tâches qui affichent du changed sans aucune diff. J'ai un souvenir douloureux d'un module qui, renvoyait simplement changed sans détails. Tout le monde s'en était accomodé, les changements étaient maîtrisés sur ce périmètre... jusqu'au jour où...

## L'option check

Autre option indispensable: le flag `-C `, check mode. Si Ansible ne peut garantir l'exécution d'un playbook, comme le ferait un Terraform, il est tout de même possible de lancer Ansible pour qu'il nous affiche ce *qu'il aurait fait* s'il n'avait pas été lancé avec le flag `-C`. 

Dans l'exemple ci-dessous, je lance deux fois le playbook, le fichier n'est jamais modifié:
![Le check mode en action](/images/blogs/01-ansible-facile/check_mod.png "Le check mode en action")

Combiné avec le diff mode `-D` c'est mon combo préféré:
* c'est rassurant de pouvoir vérifier ce qu'il devrait se passer quand on va réellement appliquer avant de le faire effectivement; et indispensable si vous espérez qu'un jour vos devs cliquent d'eux-même sur les étapes de déploiements des pipelines applicatives,
* pour travailler ses roles c'est très pratique: on peut repartir de zéro à chaque exécution, sans jamais devoir re-créer sa VM (ou serveur) avant de recommencer. Bien sûr on doit ensuite s'assurer que tout fonctionne sans le check mode, mais on peut vraiment avancer plus rapidement. Essayez, vous verrez!

**Attention**: cela ne *garanti pas* ce qu'il se passera réellement sur vos machines *au moment* ou vous allez lancer le playbook sans flag `-C`. Entre le moment où vous lancer votre check et le moment ou vous appliquerez réellement votre playbook, il peut y avoir eu des changements sur vos plateformes. Et contrairement à Terraform qui bloquerait en constatant que l'état de départ ne correspond plus à son "plan"; Ansible lui déroulera le playbook jusqu'au bout sans sourciller.

**Attention 2**: les roles ne sont pas souvent compatible avec le check mode, surtout à la première installation. Attendez-vous à devoir ajouter quelques `when: not ansible_check_mode` (voir même des `ignore_errors: {{ ansible_check_mode }}`) dans le code source des rôles que vous forkez.

```
# Check mode + diff
ansible-playbook -C -D ...
```

## Le "default callback" et ses configurations:

Quand vous lancer vos playbooks, c'est le "Default Callback" qui se charge d'afficher les résultats. Et la configuration par défaut peut s'améliorer en réglant plusieurs variables d'environnement:
* la variable `ANSIBLE_CHECK_MODE_MARKERS` pour aller avec le check mode: permet de bien indiquer si des tâches ont été lancées en check mode ou pas (encore un qui devrait être activer par défaut :/)
![check mode marker](/images/blogs/01-ansible-facile/check_mod_marker.png "Check mode marker")

- la variable `ANSIBLE_SHOW_TASK_PATH_ON_FAILURE=True` affiche le chemin de les tâches en erreur, bien pratique aussi
<!-- ![failule path](/images/blogs/01-ansible-facile/fail_path.png "Failure path") -->

<p align="center">
<img src="/images/blogs/01-ansible-facile/fail_path.png" alt="Failure path">
</p>


* la variable `ANSIBLE_CALLBACK_RESULT_FORMAT=yaml`, indispensable pour la lisibilité des debug messages
![Yaml](/images/blogs/01-ansible-facile/yaml_format.png "Yaml")

En général, dans une pipeline CI/CD, j'active toutes ces options et j'utilise également celles-ci : 
* la variable `ANSIBLE_DISPLAY_OK_HOSTS=True` masque les tâches qui sont ok
* la variable `ANSIBLE_DISPLAY_SKIPPED_HOSTS=true` masque les tâches qui sont skipped

Je ne les active que sur les étapes de check: l'objectif est de miniser la sortie pour ne voir que les changed, et avec leurs diffs bien sûr. 

```
# Check mode qui montre uniquement les changed
ANSIBLE_DISPLAY_OK_HOSTS=True \
ANSIBLE_DISPLAY_SKIPPED_HOSTS=true \
ansible-playbook -C -D ...
```

C'est avec toutes ces options que j'arrive à avoir des devs qui cliquent (presque) sereinement sur les pipelines de check d'abord, et de décider seuls d'appliquer ou non les changement qu'ils visualisent. 

Allez, une dernière pour la route : la variable `ANSIBLE_FORCE_COLOR=True`, c'est celle-ci qu'il faut activer quand vous avez une pipepline en noir et blanc dans votre outil ce CI/CD préféré. De rien :) 

Rappel : toutes ces variables sont également configurables dans un fichier fichier de configuration `ansible.cfg`. C'est même recommandé (plus propre). Voir la [doc](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/default_callback.html).

## Utiliser un autre callback

Il y a d'autres callbacks disponsibles, la liste officielles est [ici](https://github.com/ansible/ansible/tree/devel/lib/ansible/plugins/callback) mais il y en a bien d'autres.

Notamment il y a le callback [Ara](https://ara.recordsansible.org/), bien connu, qui permet d'enregistrer ses exécutions de playbooks dans une base de données et de les consulter via une interface web. Très pratique mais induit pas mal de lenteur à l'exécution (temps multiplié par 2 d'après mes mesures).

Il y en a un aussi super qui permet d'avoir une log au format Asciidoc avec un rendu ressemblant à une interface Web. Il est super mais n'a pas beacoup évolué depuis qu'il est sorti. Fin de la blague c'est le mien, mon bébé  [Caradoc](https://github.com/jseguillon/caradoc). Et si vous le trouvez cool, mettez des petites étoiles et/ou proposer des PRs ya de quoi en faire un outil formidable.

![Il est beau je vous dis](/images/blogs/01-ansible-facile/caradoc.png "Je me suis réveillé, j'avais envie de manger des logs.")

<!--TODO: nouvelle version: chainer les callbacks -->

## "Voir" ce qu'Ansible a fait sur un serveur

Ansible se connecte en ssh pour effectuer ses connexions. Et il laisse des traces bienvenues dans le syslog des machines. En fouillant les logs avec un simple grep, on peut visualiser ce qu'il s'est passé sur la machine. C'est pas fou mais ça permet au moins d'être sûr que Ansible travaille sur le bon serveur. Ou de trouver quand un fichier a été modifié par Ansible par exemple.

```
# Serveur, mon beau serveur, Ansible est-il passé par ici ?
amdin@server$ journalctl --since "1 hour ago" | grep ansible
```

## Utiliser les IAs conversationnelles et les outils IDE

On est bientôt en 2024 et les IAs prennent le pouvoir !?! Pour ma part je considère que l'outil est assez bon pour me faire gagner du temps et je ne vois donc pas de raison de m'en passser. Surtout pour Ansible qui est un puis sans fond: les listes de modules, plugins, variables, etc... c'est "inhumain". Bien sûr relisez, corrigez, etc... mais ne vous privez pas.

De même, on ne recommandera jamais assez l'installation de plugins Ansible dans votre IDE. Vous pouvez avoir de la colloration syntaxique, du lint, de la complétion, des snippets etc... même dans vim! (enfin je crois).

A la prochaine :)
<!-- ## Flag verbose

le flag -v 
(pensez à le rendre configurable dans les params de la CI ça peut servir)


(mais attention au 1er check mode => recette)

## Utiliser les LLMS et les outils IDE 
TODO: --> 

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