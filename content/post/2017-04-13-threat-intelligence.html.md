---
section: post
date: "2017-04-13"
title: "La 'Threat Intelligence' technique: Attention !"
description: "Retours sur expérience sur la réalisation d'une solution de CTI"
slug: threat-intelligence
draft: true
featured: false
image: /images/articles/2017/threat-intelligence.jpg
tags:
  - cyber
  - securite
  - bigdata
lastmod: 2017-03-29T10:25:06+02:00
---

![Threat Intelligence](/images/articles/2017/threat-intelligence.jpg)

Je travaille dans le domaine de la cybersécurité dans l'objectif de réaliser
des produits pour une entreprise fournissant des services autour de la
cybersécurité.

J'ai réalisé un certains nombre de produits visant à outiller un `SOC` (Security
  Operations Center), et un `CERT` (Computer Emergency Response Team) de cette
même société.

Lors de la réalisation des services, j'ai été confronté à la demande de pouvoir
aggréger de l'information provenant de sources diverses et variées, et de confiance
tout autant diverse et variée, afin de faciliter les intégrations dans les
équipements sécurité, ainsi que l'interrogation de dépôt de données.

C'est ainsi que je me suis penché sur ce `buzz word` que représente la `threat
intelligence`.

# Définition

> La `threat intelligence` est l’information clé définissant le contexte, les
> mécanismes et les indicateurs permettant d’anticiper les menaces à l’encontre
> du SI et de riposter de manière actionnable face à celles-ci. (Gartner)

La `threat intelligence` technique ou opérationnelle, vise à fournir des
informations techniques (IP, FQDN, Hash, etc.) appelés `Observables`, si
ceux-ci participe à un ou plusieurs scénarios malicieux, ils sont appelés
indicateurs de compromission (`IoC` en anglais).

# Chaîne de traitement

Afin de produire un `data warehouse` traditionnel, il faut mettre en place une
chaîne de traitement basée sur `ETL` (Extract / Transform / Load). La nature de
la donnée étant brute, il faut procéder à un ensemble de traitement visant à
affiner la valeur de l'information.

> A ne pas confondre avec un `data lake`, les données sont traitées **AVANT**
> l'écriture donc `data warehouse`.

Je vais concentrer mon analyse sur les problèmes et les observations réalisées
lors de la création du système.

## Collecter

### Sources Externes

#### Sources Publiques

Il existe de nombreuses sources d'informations publiques disponibles, nous en
avions identifiés `plus de 150`.

Voici les problèmes rencontrés :

  * `Accès à l'information`: fichier sur serveur WEB / FTP, page HTML,
    consommation API;
  * `Gestion des fichiers`: en outre du format de l'information (TXT, CSV, XML,
    ...), il y a aussi la compression (TAR, ZIP, etc.);
  * `Format non hétérogène`: pas de spécification unique pour gérer des listes
    d'observables. Il faut définir des stratégies de collecte paramétrables, ce
    qui rends le développement du composant de collecte plus complexe. Il existe
    bien des initiatives comme [STIX](https://oasis-open.github.io/cti-documentation/)
    ou [OpenIOC](http://www.openioc.org/) mais cela reste encore trop peu
    utilisées, c'est souvent trop lourd et le plus souvent c'est un simple CSV;
  * `Pas de garantie de disponibilité`: la source peut avoir des soucis de
    disponibilité, rendant la collecte impossible le temps de la résolution
    du/des problème(s) => rendant aveugle le temps d'indisponibilité;
  * `Questionnement sur la fiabilité`: quel niveau de confiance associé à la
    source ?
  * `Pas de contrôle d'intégrité`: il est rare de voir une signature numérique
    associée prouvant l'intégrité et la provenance du fichier collecté;
  * `Continuation de service`: c'est un service gratuit, rien ne prouve qu'il va
    le rester, et souvent les sources sont gérées en `best-effort`.

#### Sources Privées

Certaines entreprises ayant compris la difficulté à corréler les informations
publiques, vendent des `feeds` d'informations traitées et uniformisés.

Voici les problèmes rencontrés :

  * `Eparse`: peu de sources privées au final;
  * `Payantes et chères`: ces sources sont souvent `très` chères;
  * `Peu de valeur ajoutée`: il faut avouer qu'il est difficile d'avoir de
    l'information pertinente quand on est pas un fabriquant de solution de
    sécurité, ce qui a pour effet de produire des services autour de
    l'uniformisation des informations provenant de sources publiques, ainsi on
    peut s'apercevoir qu'il s'agit simplement d'un formattage de plusieurs
    sources publiques qui est vendu, compréhensible si explicité et assumé.

### Sources Internes

Il est possible aussi d'exploiter les sources d'informations internes comme :

  * `Logs des équipements`: IDS / IPS produisent des observables qui peuvent
    être qualifiés `IoC` par analyse statistique simple des fréquences;
  * `Les incidents detectés / traités`: peuvent aussi servir à alimenter en
    observables, même si ce sont des faux-positifs, il est intéressant de tracer
    ce fait;
  * `Plateforme de soumission`: il est possible de mettre en place des services
    de soumission pour transférer des fichiers suspicieux nécessitant analyse;
  * `SMTP Blackhole`: recupération des campagnes de SPAM, par l'installation de
    services mail volontairement configurés en `Open-Relay` mais controllés;
  * `HoneyPot`: pour la collecter des malwares, mais aussi des tendances de
    connexions;
  * `Quarantaine Antivirus`: les espaces de quarantaines des antivirus sont des
    viviés a malware, et permettent d'identifier des tendances d'infections.

Voici les problèmes rencontrés :

  * `Interaction de services nécessaire`: il n'est pas toujours possible d'avoir
    accès à toutes les informations d'un SI;
  * `Le HoneyPots`: la mise en place de tels dispositifs est simple à faire,
    mais garantir qu'il n'est pas utilisé comme système de rebond pour
    attaquer les machines légitimes est beaucoup plus complexe;
  * `Analyse de malwares`: pour analyser les fichiers collectés, il faut mettre
    en place tout une infrastructure ou déléguer cette tâche d'analyse à un
    service. Il faut savoir que l'`analyse dynamique` ne fonctionne pas toujours
     à cause des contre-mesures possibles du malware qui va par exemple détecter
    qu'il est dans une VM, et au mieux ne se lancera pas, ou au pire aura un
    comportement complètement différent. On aboutira rapidement à une analyse
    statique manuelle, ne pouvant pas faire face à la volumétrie;
  * `Coût de l'infrastructure de traitement`: nécessite la connaissance et
    l'administration des systèmes d'analyses pour produire des données utiles.
    Cette infrastructure est un coût masqué servant simplement à ajouter
    une ou deux étiquettes à un observable.

## Normaliser

Afin de pouvoir traiter l'information collectée, il faut la normaliser, c-à-d
la faire correspondre à un modêle de données la rendant :

  * Identifiable;
  * Comparable;
  * Historisable;
  * et surtout requêtable.

Nous avons choisi de mettre en place un dispositif de traitement distribuable
avec une gestion de `back pressure` pour gérer la pression d'intégration par
rapport au différentes vitesses que composent notre système.

Voici les problèmes rencontrés :

  * `Modèle Acteurs Distribués`: un problème est déjà difficile à résoudre dans un
    seul système, mais en plus si celui-ci est distribué c'est encore plus
    complexe;
  * `Définir un langage inter-composant`: les éléments du système doivent tous
    être capable de comprendre l'information qu'il va traiter indépendamment de
    l'étape à laquelle il est appelé;
  * `Modèle unique et extensible`: il faut être capable de transmettre
    l'information provenant des collecteurs de la manière la plus pure possible;
  * `Meta-modélisation`: en ingénierie des modèles, il faut savoir créer des
    modèles pour définir des modèles, c'est ce qui a été fait pour exprimer la
    valeur de la données, et la garder tout au lon d processus d'analyse;
  * `Sérialisation et transport`: comment transporter l'information dans le
    modèle, tout en sachant que ce modêle peut évoluer;
  * `Compression des unités de travail`: les acteurs travaillent indépendamment
    des uns des autres, à leur rythme, la file d'intégration peut contenir beaucoup
    de messages, pour optimiser cela nous avons compressé ces unités de travail;
  * `Pouvoir évoluer avec les versions`: il ne faut pas se retrouver à devoir
    faire une migration de base de données complète parce qu'on a changé un
    type de donnée, et parce qu'il est impossible de tout prévoir à la conception.

## Améliorer

Les `observables` sont à présent normalisés et prêt à être intégrés. Les
améliorations agissent sur la valeur de l'information et son contexte.

Lorsque l'on travaille sur des informations techniques pures, il est rare qu'elles
se suffisent à elles-mêmes, il faut pour cela créer des liens entre elles. La
création de ces liens est ce que nous appelons: l'amélioration.

Voici les problèmes rencontrés :

  * `Résolution DNS`: les résolutions DNS sont cruciales c'est le seul moyen
    pour obtenir d'un FQDN, le ou les IP associées. Cependant l'inverse n'est pas
    toujours possibles, il faut utiliser des services d'historisation de
    résolutions pour être capable d'avoir dans le temps les résolutions différentes.
    Il existe des fournisseurs de services pour cela, mais comme d'habitude aux vues
    de la volumétrie, ces services sont inabordables. Il faut donc faire faire
    les résolutions par le système à l'intégration;
  * `Détecter les domaines et sous-domaines`: les sources fournissent des observables
    qualifiés de domaines, mais il s'agit de sous-domaines, leur prédicat n'est pas
    faux mais il faut pouvoir le detecté, et trouver le domaine parent, sans compter
    qu'un FQDN peut aussi se cacher derrière ce domaine. Ex: `zenithar.org` mon domaine,
    mais c'est aussi un FQDN `zenithar.org` => site hébergeant mon CV;
  * `Interrogation WHOIS`: le `Whois` est très utile pour extraire les informations
    d'un domaine, mais le protocole est loin d'être respecté. Il n'a de normé
    que l'alphabet qui sert à écrire son nom, et son port de communication (TCP 43). 
    On se retrouve vite à écrire autant de parser qu'il y a des fournisseurs de 
    whois : logiquement un par TLD, soit aujourd'hui plus de 1500 ...;
  * `Décomposer des URLs`: sur le papier cela semble simple, mais il n'existe pas
    vraiment de RegExp permettant d'extraire et de valider une URL, sans compter
    IDNA et l'UNICODE;
  * `Géolocalisation des IPs`: Attention c'est pas un système fiable, il ne faut
    pas avoir une confiance aveugle, et dire : 'les chinois nous attaquent par 
    votre voisin utilise un noeud TOR de sortie en chine ...';
  * `Détecter les erreurs de classification`: les sources n'ont pas toujours une
    confiance absolue, il faut avouer que ce sont des traitements informatiques
    qui font le travail, scripts développés par un humain qui peut faire des
    erreurs. Nous avions souvent le cas d'URL extraite depuis le code HTML, qui
    contenaient des simples quotes en fin d'URL. C'est une problèmatique récurrente
    liée au `web scrapping`; Autre cas les sandboxes qui renvoient les appels de fonction
    sous le format suivant `getlogicalprocessorinformation@kernel32.dll`, format qui
    peut être considéré comme une adresse email syntaxiquement valide !
  * `Améliorer vite !`: il ne faut pas ajouter une latence de traitement à
    l'information, si nous mettons plus de 24h à traiter un évènement sur 15
    minutes, l'intérêt peut être du coup limité, et générer dans le pire des cas
    de la fausse information. Internet est vivant, il n'attends pas que vous
    ayez fini vos traitements pour muter / évoluer (ex: bail IP dynamique =>
    assignation d'une étiquette attaquant à l'IP alors que la personne ne
    possède plus le bail );
  * `Gestion du cache distribué`: un observable déjà augmenté ne doit pas faire
    perdre du temps à toute l'intégration, il doit être ignoré s'il vient
    d'être traité. Cependant l'introduction d'un cache peut aussi faire qu'un
    changement passe inaperçu car la valeur a changée avant l'éviction du cache.

## Stocker

Le stockage des `observables` est aussi très important puisque l'objectif est
de pouvoir construire un `data warehouse`, contenant les informations traitées
par notre infrastructure de `data wrangling`.

Voici les problèmes rencontrés :

  * `Volumétrie`: il s'agit de traiter des millions d'informations par jour,
    plusieurs fois par jour. Un parle bien de `Big Data`, ce n'est pas un ensemble
    de macros Excel qui passe sur un classeur à 10 000 lignes ... (vu sur internet);
  * `Stockage Hybride`: pouvoir requêter de manière complexe tout en gardant
    de performance d'écriture acceptable;
  * `Big Data == Big Infra`: parce que 3 Raspberry PI en réseau ne suffisent pas;
  * `Modification parallèle et incrémentale`: La nature distribuée du travail fait
    que l'ordre des traitements sur la donnée finale ne doit faire qu'améliorer
    la valeur de celle-ci, une opération ne peut pas dévaluer un observable parce
    qu'il s'est produit une erreur;
  * `Amlérioration par fusion`: les observables sont améliorés par le système de base 
    de données, pas de lecture => fusion => écriture;
  * `Version community VS enterprise`: beaucoup de produit propose gratuitement
    leurs outils n'ayant que peu de différences sur le papier, mais la réalité
    est tout autre, nous avons eu des problèmes liés à la volumétrie seule,
    cette limitation n'est pas indiquée clairement. De plus les versions
    `enterprise` sont inaccessibles financierement parlant ...;
  * `Suivi de version en version community`: en utilisant la version grand public
    vous n'êtes pas à l'abrit que certaines fonctionnalités disparaissent pour
    être intégrées en version enterprise uniquement. Tout en sachant qu'il
    n'existe pas d'alternative viable, ce qui s'appelle du `hold-up
     commercial`;
  * `Mr Bricolage`: les mouvements de fonctionnalités, quand on a pas les moyens
    d'acheter, cela donne lieu à des bricolages architecturaux, on ajoute un
    composant pour pallier à la disparition, on comble avec d'autres outils.
    Augmentant la complexité de l'infrastructure et donc les probabilités de
    pannes, bugs, etc;
  * `Historisation des modifications`: c'est une fonctionnalité très importante 
    mais elle occasionne un impact très important en infrastucture. 
    Nous avions utlisé une stratégie CoW (Copy on Write) pour garantir l'intégrité,
    mais est arrivé les problèmatiques de rétention de données. Nous avons 
    choisi de garder la dernière version de l'observable.
  * `Sauvegarde`: Plus il y a de données plus la sauvegarde est importante.

## Analyser

Le dépot de données est construit, il faut maintenant l'exploiter.

Voici les problèmes rencontrés :

  * `Data Scientist c'est un métier`: il est très difficile d'analyser des
    données sans formation adaptée. Nous avons construis un dépôt contenant des
    millions d'enregistrements reliés dans un hypergrahe, ces données sont
    exploitables noeuds par noeuds, mais il faut être capable de concevoir des
    algorithmes exploitant la structure de l'information;
  * `Peu de théorisation`: nous avons choisi d'utiliser une structure graphe
    pour stocker nos observables, il existe des outils mathématiques pour analyser
    des graphes, mais ils sont encore à l'état de thèse, ou utilisés pour des cas
    précis (moteur de recommendation, etc.);
  * `Taux de recopie entre les sources`: plus on augmente le nombre de sources
    plus on s'aperçoit que les sources se recopient entre-elles, visible par
    analyse des clusters;
  * `Gestion de la confidentialité des observables`: effectivement les
    observables sont liés à l'analyse du malware qui lui est contenu dans un
    document parfois officiel, faut-il par extension rendre confidentiel ces
    `observables` ? ou séparer les niveaux de confidentialité ? Il existe un
    protocole appelé le [TLP](http://blog.marcfredericgomez.fr/ssi-traffic-light-protocol/) (`Traffic Light Protocol`), indiquant le niveau de
    confidentialité à respecter pour l'échange de l'information. Il définit 4 niveaux de
    confidentialité, mais n'explique pas les logiques de régressions entre un
    observable qualifié par deux entités différentes à des niveaux différents:
    on pourrait dire que la priorité est la confidentialité, donc plus c'est haut
    mieux c'est. Mais, en mon sens, l'objectif de la `threat intelligence` est le
    partage, contrôlé certes, où encore un fois la confiance rentre en jeu; Si
    une source publique fiable indique un TLP White (information publique), mais
    qu'une source privée peu fiable indique Amber (diffusion restreinte), pourquoi appliquer
    Amber ? Souvent on invoque le GBS (Gros Bon Sens);
  * `Scorer un observable`: la volumétrie fait qu'il faut prioriser les analyses,
    il faut pour cela produire un score permettant d'identifier les observables
    importants des moins importants. Cependant, un calcul trop simple risque de
    ne pas remonter les bonnes interprétations, et trop compliqué sera difficile
    à comprendre, donc expliquer et maintenir;

## Exploiter

Nous avons analysé l'information pour lui donner sa valeur maximale, il faut
maintenant passer à l'exploitation technique.

Voici les problèmes rencontrés :

  * `Capacité des produits`: nous pouvons exporter des `data-sets` depuis la
    base de données, mais malheureusement les outils de surveillance ne sont
    souvent pas capables de contenir l'ensemble de l'export;
  * `Confiance déléguée`: nous avons observé un transfert de confiance au produit
    de collecte et d'analyse mais il faut toujours avoir un oeil critique sur
    la source de l'information. Tout traitement informatique ne peut remplacer
    la capacité de jugement et d'adaptabilité d'un être humain (pour le moment);
  * `Supervision`: ce système est distribué sur plusieurs machines qui doivent
    être supervisés sur le plan matériel, réseaux, mais aussi applicatif: absence
    de collecte, collecte identique depuis quelques jours, pas de résolutions DNS,
    etc;
  * `Couplage avec le réseau important`: le moindre problème a un impact
    sur le système, le rendant indisponible, produisant du retard d'intégration,
     etc.

# Conclusion

Voici une liste non-exhaustive des problèmes et réflexions qu'il faut prendre en
concidération; si vous envisagez de construire une telle solution. Un système
de `Cyber Threat Intelligence` technique nécessite des moyens techniques,
financiers, mais aussi en ressources humaines importants.

Depuis 3 ans, il existe de nombreuses solutions OpenSource, éliminant la charge
de développement de la solution, tout en gardant une charge pour adapter, mais
ne supprimera en aucun cas le besoin en infrastructure.

> Je n'irai pas jusqu'à justifier les prix pratiqués par des solutions commerciales
> mais il faut comprendre pourquoi c'est aussi cher.
>
> C'est le commerce de l'information ! On ne vends pas le produit mais l'information
> qu'il a traité.
