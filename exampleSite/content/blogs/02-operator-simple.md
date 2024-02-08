---
title: "Comprendre les Operators Kubernetes rapidement"
date: 2024-02-07T00:02:21+01:00
draft: false
github_link: "https://github.com/jseguillon"
author: "jseguillon"
tags:
  - Operator
  - Kubernetes
  - Explications
  - Simplifications
image: /images/post2.png
images: 
  - /images/post2.png

description: 'Une explication simplifiée de la pattern "Operator" pour Kubernetes'
toc:
---

C'est une histoire qu'on raconte pour faire peur aux enfants mais qui pourtant fait frémir d'horreur même les plus chevronné(e)s. L'histoire d'une ingénieure qui voulait héberger une base de données dans un cluster Kubernetes. Après avoir scrupuleusement lu la doc technique, elle se lança hardiement dans la production de fichiers YAMLs pour porter son hébergement dans le cluster. Connaissant bien ses Pods, Deployements, Services et autre objets Kubernetes, la tâche ne semblait pas si ardue. Mais elle dû produire tant et tant de fichiers YAMLs, de templates Helm et de Ansible que son moral finit par s'écrouler et elle choisit de se reconvertir en agente immobilière. Une fin tragique.

![scared](/images/blogs/02-operator-simple/scared.gif)

Heureusement ça ne se passe pas exactement comme ça dans le monde Kubernetes.

## Pattern Operator: l'utilité de la chose

Si les personnes en charge de fournir des services aiment tant Kubernetes c'est qu'il y a une solution élégante à ce problème : la pattern Operator. Et ça rend le déploiement de services complexes encore plus simples que sur des VMs.

Je vous explique: en général si on veut déployer une base de données haute dispo (ou équivalente), un système de fichier distribué, un stockage objet etc..., dans des VMs et sans Kubernetes, on va devoir sortir l'artillerie lourde. Comprendre : des lignes de shell/Ansible/Pulumi/whatever, qui vont devoir déployer la solution (paquets/fichier de confs), potentiellement sur des noeuds de différents OS (aïe). Ça se fait mais je vous assure c'est pas évident (et c'est coûteux). A coder manuellement, on drift du manuel d'install officiel, qui lui même se met à jour au gré des versions qu'on doit reporter, bref c'est pas la joie.

Dans Kubernetes c'est différent. Comme les APIs fournissent des objets standard qui décrivent les unités d'exécution, l'équilibrage de charge, la réplication, le scaling, etc... on peut écrire *un programme* qui sait déployer correctement un service complexe dans les règles de l'art. Et peu importe les briques techniques ou les OS qui seront derrière, l'ensemble des mécanismes dont le service a besoin pour fonctionner seront garantis par l'API et les objets que le dit programme aura créé. Ce programme, c'est l'Operator.

{{< picture "blogs/02-operator-simple/operator-base.svg" "Description d'un opérateur" >}}

## Operator: la magie c'est l'API

Si vous avez bien suivi, on parle d'un programme qui crée des objets sur l'API Kubernetes, un peu comme si vous poussiez des YAMLs de façons automatisée. Et là vous me dîtes: si je dois coder un "Operator", ça va pas être plus simple que de faire du Ansible ou du Helm.

La magie c'est que vous n'avez probablement pas besoin de coder votre Operator: il existe déjà ! Attirés par l'abstraction fournie par l'API Kubernetes, la plupart des fournisseurs de technologies vous offrent leur Operator. Une forme de packaging ultime qui vous garantit qu'un service aussi complexe qu'il soit, sera déployé sans soucis dans n'importe quel Kubernetes correctement configuré.

Quelques exemples:
* Une base mongo, par l'Operateur Percona :

```yaml
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDB
spec:
  replsets:
  - name: rs0
    size: 5
...
```

* Un cluster Cephs, par l'Operator Rook :

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
spec:
  mon:
    count: 3
...
```

Il y en a pour tous les goûts, une liste de référence ici : https://operatorhub.io/ mais comme chacun peut écrire son propre Operator, il y en d'autres qui traînent.

## Custom Resource et Custom Resource Definitions

Bien sûr de base il n'existe pas d'objet `PerconaServerMongoDB` tel que présenté dans l'exemple précédent, au sein d'un cluster Kubernetes standard. Pour pouvoir créer un tel objet, il va falloir enrichir l'API Kubernetes. On va donc accompagner l'Operator d'un contrat d'API adhoc, créé pour l'occasion, grâce au mécanismes de `Custom Resource Definition`, un simple schéma json poussé au moment de l'installation de l'Operator.

Exemple pour notre définition d'une base de données mongo :

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
spec:
  group: psmdb.percona.com
  names:
    schema:
      openAPIV3Schema:
        properties:
...
          spec:
            properties:
              replsets:
...
                  properties:
                     size:
                       format: int32
                        type: integer
...
```

En installant un Operator, on enrichit donc l'API Kubernetes de nouveaux objets, dont la signature est connue de l'Operator. Ces objets sont validés par l'API Kubernetes et stockés dans le backend Kubernetes (base etcd ou postresql en général). C'est très pratique parce qu'on délègue la validation et le stockage des instanciations de ces nouveaux objets, les `Custom Resources` (notre instance de `PerconaServerMongoDB` avec ses 5 replicas), à Kubernetes. Ça allège le code et ça permet également de bénéficier, pour ces nouvelles ressources, du support natif de fonctionnalités Kubernetes, tel que RBAC(=qui a le droit de créer/modifier une base mongo), nativement supporté pour les `Custom Resources` comme toutes resources classique de Kubernetes.

{{< picture "blogs/02-operator-simple/CrCrd.png" "Cr vs CRD" >}}

## Controller et boucle de réconciliation 

Pour le moment, on a parlé d'Operator comme étant le programme qui lit un YAML de type Custome Resource. Ce n'est pas exact: ce programme est en fait nommé `Controller` et il *fait partie* de l'Operator. 

Le `Controller` réconcilie votre définition de ressource avec l'état du cluster : 
  * si vous modifier la définition de votre bdd, le `Controller` va prendre en charge la mise à jour de la config dans les objets Kubernetes
  * si vous modifer les objets Kubernetes créés par le `Controller`, celui-ci va écraser vos modifications

{{< picture "blogs/02-operator-simple/loop.png" "La boucle de réconciliation" >}}

Ce ce qu'on appelle "la boucle de réconciliation". Bêtement le `Controller` de votre Operator fera absolument tout ce qu'il peut pour que votre cluster Kubernetes reflète très exactement les spécifications que vous avez demandé pour une des `Custom Resource` qu'il contrôle.

A noter que l'Operator peut porter des fonctionnalités complexes dans sa boucle de réconciliation : notamment pour ordonnancer les mises à jour de versions. Changer de version mongo en ne faisant rien d'autre que de changer un champs version dans un YAML, avouons que ça rend les choses plus simple.

## Le plein de Custom Resources

Dans le schéma ci-dessus, j'ai mis un "s" à CRD. Effectivement les Operators ont tendance à installer plus d'un CRDs. Parfois c'est juste une manière d'organiser en interne le stockage d'information de l'Operator (exemple: vous saisissez un objet de type "pool d'IP" et l'Operator trace des objets supplémentaires de type "ip réservée"). Mais souvent ça apporte des fonctionnalités supplémentaires. Par exemple, dans notre Operator mongodb, il y a des objets qui vous pouvez pousser pour demander un snapshot immédiat, planifier un job de backup régulier, etc..

Exemple pour lancer un backup :

```yaml
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDBBackup
spec:
  clusterName: my-cluster-name
  storageName: s3-us-west
```

Ça facilite vraiment l'exploitation d'un produit au sein : au lieu de chercher à se connecter et lancer des commandes shell, on peut juste pousser le YAML et attendre le retour. Car les `Customs Resources` contiennent tous un champ `status` : dans notre exemple on pourra vérifier si le backup s'est bien passé et consulter les erreurs si il y en a. Directement dans l'objet backup, sans éplucher les logs des Pods Kubernetes. Cette interface unique pour instancier et exploiter un service complexe sous Kubernetes est vraiment très confortable pour les fournisseurs de plateformes.

## L'envers du décor

Le monde n'étant pas parfait, voici deux limites à prendre en compte lorsqu'on utilise les Operator. 

### Le format des CRs

Premièrement, la complexité des formats CRs. Implicitement vous aurez peut-être compris que les CRs mixent en fait généralement deux choses :
  * l'hébergement du service dans Kubernetes (combien de quels pods, services etc...)
  * la configuration du service lui même (fichiers de configs de mongo dans notre exemple)

Ce mélange des genres ne rend pas la configuration des CRs évidentes. Surtout lorsque l'on cherche une option particulière qu'on injecter dans les fichiers de confs, on peut vite se perdre... ou ne pas trouver. Parce qu'évidemment, comme l'Operator est un soft lui-même, si l'option de conf que vous cherchez n'existe pas dans le CR, il ne vous restera plus qu'à ouvrir un ticket sur le dépôt du projet et croiser les doigts. A moins que vous ne sachiez coder en Go bien sûr.

### Les releases

On rappelle : l'Operator (enfin son `Controller`) est un programme. Qui dit programme dit "build", ou au moins "release". Et là ça peut coincer. En bon dev, les équipes en charge de votre Operator préféré vont faire simple : suivre les mises à jour Kubernetes au plus près. Mais vous ? Imaginons que vous n'ayez pas fait de mise à jour Kubernetes depuis quelques versions, et que vous avez une CVE critique sur un de vos Operators ou produits qu'ils déploient. Vous cherchez le correctif et là c'est le drame : votre Operator corrige la CVE mais uniquement sur des versions au-dessus vos propres versions de Kubernetes ou Operator. De quoi s'offrir quelques frayeurs.

Bon là c'est en partie de votre "faute", mais imaginons encore : vous êtes au top des mises à jours, autant Kubernetes que les Operators, et vous avez la même CVE. Vous faîtes la mise à jour et bim votre Operator ne répond plus comme avant. La régression est bien détectée et un fix est déjà proposé mais voilà, il va falloir attendre que quelqu'un fasse la release et publie une nouvelle image. Et en attendant vous êtes bloqués alors que la solution est devant vos yeux. Frustrant.

Pour traiter ces deux problèmes : forkez les dépôts git et prenez en charge vous-même le build des Operators que vous utilisez. Ça vous permettra de résoudre ce genre de problèmes par un bon cherry-pick ou merge.

## A vous de jouer

Vous savez tout, il est temps maintenant de vous amuser. Choisissez une technologie que vous avez envie de tester, installez son Operator et profitez de services prêts à l'emploi, déployés à la vitesse de l'éclair. Une bonne façon d'apprécier la qualité de l'écosystème qui gravite autour de Kubernetes.
