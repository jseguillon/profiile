---
title: "Ansible, une meilleure expérience est possible."
date: 2023-11-28T00:02:21+01:00
draft: false
github_link: "https://github.com/jseguillon"
author: "jseguillon"
tags:
  - Ansible
  - Débuter
  - Tips
image: /images/post.jpg
images: 
  - /images/post.jpg

description: "Quelques astuces pour se faciliter la vie avec Ansible"
toc:
---


Ansible a mauvaise réputation en terme d'expérience utilisateur:
* que ce soit pour l'ops/sre qui peine parfois à produire ses YAMLs
* ou pour le dev qui voudrait au moins pouvoir lancer quelques pipelines lui-même mais qui ne comprend rien aux logs et qui a peur de faire une bêtise.


Ce billet de blog, tout premier pour moi, est là pour donner quelques pistes pour se rendre la vie plus belle quand on utilise Ansible. Let's go!

*Note*: si vous n'avez jamais lancer de playbooks Ansible et que vous recherchez des ressources, commencez par la fin de l'article et visitez le [site de Stéphane](https://blog.stephane-robert.info/).

## L'option diff


On commence par une option indispensable de la commande `ansible-playbook`: l'option `-D`, diff. Cette option va vous permettre de visualiser les modifications effectuées par une tâche Ansible. Une démo valant mieux que 100 mots, voici un avant/après option `-D`.


![Le diff mode en action](/images/blogs/01-ansible-facile/diff_mod.png "Le diff mode en action")


Vous l'aurez compris, l'option nous permet de voir au-délà du simple statut `[changed]`, les modifications **réellement apportées** par Ansible. Si vous voulez mon avis, cette option devrait être activée par défaut dans Ansible...


```
# Toujours mettre le flag -D
ansible-playbook -D ...
```


**Note** : La *plupart* des modules fonctionnent proprement avec le diff mode. Mais faites la chasse aux tâches qui affichent du changed sans aucune diff ! J'ai un souvenir douloureux d'un module qui renvoyait simplement `[changed]` sans détails. Tout le monde s'en était accommodé, les changements étaient maîtrisés sur ce périmètre... jusqu'au jour où...


## L'option check


Autre option indispensable: le flag `-C `, check mode. Si Ansible ne peut garantir l'exécution d'un playbook, comme le ferait un Terraform, il est tout de même possible de lancer Ansible pour qu'il nous affiche ce *qu'il aurait fait* s'il n'avait pas été lancé avec le flag `-C`.


Dans l'exemple ci-dessous, je lance deux fois le playbook, le fichier n'est jamais modifié:
![Le check mode en action](/images/blogs/01-ansible-facile/check_mod.png "Le check mode en action")


Utilisé avec le diff mode `-D` c'est mon combo préféré:
* c'est rassurant de pouvoir vérifier ce qu'il *devrait* se passer avant d'appliquer; et indispensable si vous espérez qu'un jour vos devs lancent d'eux-même l'exécution des playbooks qui les concernent,
* pour travailler ses rôles c'est très pratique: on peut "repartir de zéro" à chaque exécution (en fait on ne fait rien), sans jamais devoir re-créer sa VM (ou serveur) avant de recommencer. Bien sûr on doit, à la fin, s'assurer que tout fonctionne sans le check mode, mais on peut vraiment avancer plus rapidement. Essayez, vous verrez!


```
# Check mode + diff le combo parfait
# Pour se rassurer ou pour dev plus vite
ansible-playbook -C -D ...
```


**Attention**: cela ne *garanti pas* ce qu'il se passera réellement sur vos machines *au moment* ou vous allez lancer le playbook sans flag `-C`. Entre le moment où vous lancer votre check et le moment ou vous appliquez réellement votre playbook, il peut y avoir eu des changements sur vos plateformes (ou sur une dépendance, un nouveau paquet publié sur un dépôt par exemple).


**Attention 2**: les rôles ne sont pas si souvent compatibles avec le check mode, surtout lorsqu'il n'a pas encore été lancé. Attendez-vous à devoir ajouter quelques `when: not ansible_check_mode` (voir même des `ignore_errors: {{ ansible_check_mode }}`) dans le code source des rôles que vous forkez.


## Le "default callback" et ses configurations


Quand vous lancer vos playbooks, c'est le plugin "default callback" qui se charge d'afficher les résultats. Et la configuration par défaut peut s'améliorer en réglant quelques configurations.

### `ANSIBLE_CHECK_MODE_MARKERS` 

pour aller avec le check mode: permet de bien indiquer si des tâches ont été lancées en check mode ou pas

![check mode marker](/images/blogs/01-ansible-facile/check_mod_marker.png "Check mode marker")


### `ANSIBLE_SHOW_TASK_PATH_ON_FAILURE` 

affiche le chemin et la ligne de les tâches en erreur, bien pratique aussi

![failule path](/images/blogs/01-ansible-facile/fail_path.png "Failure path")


### `ANSIBLE_CALLBACK_RESULT_FORMAT`

permet de changer le format de sortie de json vers yaml, indispensable pour la lisibilité

![Yaml](/images/blogs/01-ansible-facile/yaml_format.png "Yaml")


### `ANSIBLE_DISPLAY_OK_HOST` et `ANSIBLE_DISPLAY_SKIPPED_HOSTS`

En général, dans une pipeline CI/CD, j'active toutes ces options et j'utilise également celles-ci :
* la variable `ANSIBLE_DISPLAY_OK_HOSTS=False` masque les tâches qui sont ok
* la variable `ANSIBLE_DISPLAY_SKIPPED_HOSTS=False` masque les tâches qui sont skipped


Je ne les active que sur les étapes de check: l'objectif est de miniser la sortie pour ne voir que les changed, et avec leurs diffs bien sûr.

Dans la démo ci-dessous, il y a plusieurs ok et skipped mais on voit uniquement l'unique diff, c'est top !

![Displays](/images/blogs/01-ansible-facile/displays.png "Displays")

C'est avec toutes ces options que j'arrive à avoir des devs qui cliquent (presque) sereinement sur les pipelines de check d'abord, et de décider seuls d'appliquer ou non les changements qu'ils visualisent. 

```
# Check mode qui montre uniquement les changed
ANSIBLE_DISPLAY_OK_HOSTS=False \
ANSIBLE_DISPLAY_SKIPPED_HOSTS=False \
ansible-playbook -C -D ...
```

### Une dernière pour la route 

la variable `ANSIBLE_FORCE_COLOR=True`, c'est celle-ci qu'il faut activer quand vous avez une pipepline en noir et blanc dans votre outil ce CI/CD préféré.

**! Note importante !** toutes ces variables sont également configurables dans un fichier fichier de configuration `ansible.cfg`. C'est même recommandé (plus propre). Voir la [doc](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/default_callback.html).


## Ajouter un autre callback


La doc Ansible nous explique :
* que le default callback plugin est un plugin de type "stdout"
* qu'on ne peut utiliser qu'un seul "callback plugin stdout" à la fois


Ca semble triste mais heureusement, Ansible a prévu [d'autres types](https://docs.ansible.com/ansible/latest/plugins/callback.html) de callback plugins
![Types de callback plugins](/images/blogs/01-ansible-facile/ansible_doc_callback_plugins.png "Types de callback plugins")


Dans la catégorie "notifications", il y a notamment le callback [Ara](https://ara.recordsansible.org/), qui permet d'enregistrer ses exécutions de playbooks dans une base de données et de les consulter via une interface web. Très pratique mais induit pas mal de lenteur à l'exécution (temps multiplié par 2 d'après mes mesures, il y a un an; ça c'est peut-être amélioré).


Il y en a un aussi super, catégorie "aggregator", qui permet d'avoir une log au format Asciidoc et pourtant un rendu similaire à une interface Web. Il est incroyable mais n'a pas beacoup évolué depuis qu'il est sorti. Fin de la blague c'est le mien, mon bébé [Caradoc](https://github.com/jseguillon/caradoc). Et si vous le trouvez cool, mettez des petites étoiles et/ou proposer des PRs ya de quoi en faire un outil formidable. Et il y même une [démo en ligne](https://jseguillon.github.io/caradoc/)

[!["Les logs, c'est la vie."](/images/blogs/01-ansible-facile/caradoc.png "Les logs, c'est la vie.")](https://jseguillon.github.io/caradoc/)


EDIT: Je rajoute un callback qui à l'air vraiment bien. Je ne l'ai pas testé mais comme c'est fait par un français je vous le signale sans même réfléchir :) => ["ansible_stdout_compact_logger"](https://github.com/octplane/ansible_stdout_compact_logger) 

## "Voir" ce qu'Ansible a fait sur un serveur


Ansible se connecte en ssh pour effectuer ses connexions. Et il laisse des traces bienvenues dans le syslog des machines. En fouillant les logs avec un simple grep, on peut visualiser ce qu'il s'est passé sur la machine. C'est pas fou mais ça permet au moins d'être sûr que Ansible travaille sur le bon serveur. Ou de trouver quand un fichier a été modifié par Ansible par exemple.


```
# Serveur, mon beau serveur, Ansible est-il passé par ici ?
amdin@server$ journalctl --since "1 hour ago" | grep ansible
```


## Utiliser les IAs conversationnelles et les outils IDE


On est bientôt en 2024 et les IAs prennent le pouvoir !?! Pour ma part je considère que l'outil est assez bon pour me faire gagner du temps et je ne vois donc pas de raison de m'en passer. Surtout pour Ansible qui est un puis sans fond: les listes de modules, plugins, variables, etc... c'est "inhumain". L'IA est un super outil pour ça. Bien sûr relisez, corrigez, testez ce que produisent les IAs... mais ne vous en privez pas :)


De même, on ne recommandera jamais assez l'installation de plugins Ansible dans votre IDE. Vous pouvez avoir de la coloration syntaxique, du lint, de la complétion, des snippets etc... même dans vim ! Ci-dessous l'extension Vscode en action avec Lint qui pleure et la complétion qui fait le job.


![Vs code Ansible extension](/images/blogs/01-ansible-facile/vs_code_ansible.png "Vs code Ansible extension")


La page qui listes les plugins officiels est [ici](https://ansible.readthedocs.io/projects/language-server/)


## Consulter le site de Stéphane Robert


Stéphane produit un contenu exhaustif et de grande qualité sur Ansible, avec une partie blog et une partie formation devops. Si ce n'est déjà fait, consultez d'urgence son [site](https://blog.stephane-robert.info/), il est rempli de bon conseils !


## La suite au prochain numéro

Pour finir un grand merci à Quentin Joly pour sa relecture et ses conseils. N'hésitez pas à consulter son excellent [blog](https://une-tasse-de.cafe/), il y parle notamment d'infrastructures et d'outils. Des billets à déguster avec une bonne tasse de café bien sûr !


N'hésitez pas à me faire vos retours ou commentaires, directement sur Twitter (ooops "X" on dit désormais...), je suis friand de feedback :)
