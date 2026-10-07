# Déploiement automatisé d'une infrastructure multiservice avec Ansible

> **Contexte**
>
> NETSYS doit pouvoir reconstruire et exploiter son infrastructure sans dépendre d'une intervention manuelle. Cinq serveurs Debian portent cinq services distincts : serveur web, serveur DNS, serveur SFTP, base de données MariaDB et supervision.
>
> Le socle de ces machines est décrit dans le document précédent : une image système unique préparée, convertie en modèle, puis clonée cinq fois, avec une configuration injectée au premier démarrage.
>
> Le présent document couvre l'étape suivante. L'installation et la configuration des services sont décrites dans des rôles Ansible, puis appliquées par un playbook unique.
>
> La contrainte directrice tient en une phrase : ce qui n'est pas dans le dépôt n'existe pas. Une configuration posée à la main dans une machine n'est pas une configuration, c'est un état qui disparaîtra au prochain redéploiement.

- **Date** : 7 octobre 2026
- **Contrôleur** : poste d'administration, Ansible core 2.20
- **Machines cibles** : cinq, Debian 13, adressées de 192.168.10.20 à 192.168.10.24
- **Rôles** : six
- **Exigence de recette** : la seconde exécution ne produit aucun changement

---

## 1. Objectif et périmètre

Le projet doit permettre de passer d'un ensemble de machines vides à une infrastructure complète en une seule commande, de façon répétable.

Le périmètre s'arrête à la mise en service des cinq services et à la vérification de leur fonctionnement. La sauvegarde centralisée et la supervision avancée ne sont pas traitées ici.

Deux exigences structurent l'ensemble du travail.

La première est la **standardisation** : chaque machine reçoit un socle identique, décrit une seule fois, puis son service propre. Aucune configuration n'est saisie manuellement sur une machine.

La seconde est la **reproductibilité vérifiable**. Elle se mesure, elle ne se déclare pas. Le projet est donc exécuté deux fois de suite, et la seconde exécution doit afficher zéro modification.

## 2. Architecture du dépôt

```
ansible-project/
├── ansible.cfg
├── site.yml
├── inventory/
│   └── hosts.ini
├── group_vars/
│   └── all.yml
└── roles/
    ├── common/       tasks/  templates/
    ├── web/          tasks/  templates/  handlers/
    ├── dns/          tasks/  templates/  handlers/
    ├── sftp/         tasks/  templates/  handlers/
    ├── database/     tasks/
    └── monitoring/   tasks/
```

Trois niveaux, trois responsabilités distinctes.

L'**inventaire** décrit les machines et la façon de s'y connecter. C'est la seule source de vérité sur l'adressage.

Les **variables** décrivent ce qui caractérise le projet : le nom du site, le domaine DNS, le compte SFTP, le nom de la base de données. Elles sont rassemblées dans un fichier unique, ce qui évite qu'une même valeur soit écrite à plusieurs endroits.

Les **rôles** décrivent l'état attendu de chaque service. Un rôle contient ses tâches, ses gabarits et ses déclencheurs, et ne dépend d'aucun autre rôle.

Le fichier `ansible.cfg` évite de répéter les options sur chaque commande. Il déclare l'emplacement de l'inventaire, impose la découverte silencieuse de l'interpréteur Python sur les machines cibles et désactive la création de fichiers de reprise.

## 3. Inventaire et accès

Chaque machine dispose de son propre groupe, ce qui permet d'adresser un service sans toucher aux autres.

```ini
[web]
web01 ansible_host=192.168.10.20

[dns]
dns01 ansible_host=192.168.10.21

[sftp]
srv01 ansible_host=192.168.10.22

[database]
db01 ansible_host=192.168.10.23

[monitoring]
mon01 ansible_host=192.168.10.24

[all:vars]
ansible_user=admin
ansible_ssh_common_args='-o StrictHostKeyChecking=accept-new -o ProxyJump=root@192.168.1.150'
```

Le compte d'administration `admin` est commun aux cinq machines. L'authentification se fait par clé, jamais par mot de passe : la clé publique a été injectée dans le système au moment du provisionnement, ce qui rend la chaîne complète sans saisie interactive.

**Point de vigilance : accès depuis le poste d'administration.**

Dans le laboratoire d'origine, le poste d'administration et les machines cibles partagent le même réseau. Ce n'est pas le cas ici : le poste est sur le réseau domestique, et les machines sont derrière un pont interne à l'hyperviseur, sans route entre les deux.

Sans correction, chaque tentative de connexion reste en attente puis échoue. La solution retenue est une option de saut appliquée à tous les hôtes : le trafic sort par l'hyperviseur, qui voit les deux réseaux. Elle est déclarée une seule fois dans l'inventaire, donc invisible dans les commandes d'exécution.

L'option d'acceptation des clés inconnues est jointe pour la même raison : après un reclonage, les empreintes des machines sont neuves et une validation interactive bloquerait une exécution automatisée.

## 4. Orchestration

L'orchestration tient dans un seul fichier, `site.yml`, et se lit comme un sommaire.

```yaml
---
- name: Configuration commune
  hosts: all
  become: true
  roles: [common]

- name: Web
  hosts: web
  become: true
  roles: [web]

- name: DNS
  hosts: dns
  become: true
  roles: [dns]

- name: SFTP
  hosts: sftp
  become: true
  roles: [sftp]

- name: Base de donnees
  hosts: database
  become: true
  roles: [database]

- name: Supervision
  hosts: monitoring
  become: true
  roles: [monitoring]
```

Le premier jeu s'applique à toutes les machines et pose le socle. Les cinq suivants ciblent chacun un groupe, donc une seule machine, et installent un service.

L'élévation de privilèges est déclarée au niveau de chaque jeu. Elle est nécessaire pour installer des paquets, écrire dans la configuration système et manipuler les services.

Cette organisation présente un intérêt pratique immédiat : on peut rejouer une partie précise du déploiement sans toucher au reste. Limiter l'exécution à une machine revient à la cibler par son nom.

```bash
ansible-playbook site.yml --limit web01
```

C'est ce qui a permis de développer et de valider un rôle à la fois, sur une seule machine, avant de lancer l'exécution complète.

## 5. Les six rôles

| Rôle | Cible | Service | Points remarquables |
|---|---|---|---|
| `common` | toutes | socle | paquets de base, nom d'hôte, fuseau horaire, fichier de correspondance noms et adresses |
| `web` | WEB01 | Apache | page servie depuis un gabarit, redémarrage conditionnel |
| `dns` | DNS01 | Bind9 | zone locale, trois gabarits, redémarrage conditionnel |
| `sftp` | SRV01 | SSH / SFTP | compte de service, cloisonnement, validation de configuration |
| `database` | DB01 | MariaDB | base et compte applicatif, mot de passe posé à la création |
| `monitoring` | MON01 | exportateur de métriques | attente active de l'écoute réseau |

### 5.1 Socle commun

Le rôle `common` installe les paquets de base, dont l'agent de communication entre la machine invitée et l'hyperviseur, impose le fuseau horaire et génère le fichier de correspondance entre noms et adresses à partir de l'inventaire.

Ce fichier est produit par un gabarit qui parcourt la liste des machines du projet. Chaque machine peut donc joindre les autres par leur nom, sans dépendre d'un service de résolution externe.

Le rafraîchissement de la liste des paquets est borné dans le temps. Sans cette borne, la tâche se relancerait à chaque exécution et produirait un changement inutile.

### 5.2 Serveur web

Apache est installé, activé, et sert une page générée à partir du gabarit du projet. La page affiche le nom du site et le nom de la machine qui répond, ce qui permet de vérifier en une requête que la bonne machine a répondu.

Le gabarit est associé à un déclencheur : si le contenu change, un redémarrage du service est programmé. Si le contenu est identique, rien ne se passe. C'est ce mécanisme qui rend la seconde exécution silencieuse.

### 5.3 Serveur DNS

Bind9 est configuré avec une zone locale et un enregistrement qui pointe vers le serveur web. Trois gabarits sont déployés : les options générales, la déclaration de la zone, et le contenu de la zone.

Trois points sont critiques.

L'écoute réseau est restreinte à l'adresse de la machine et à la boucle locale. Par défaut, le service écoute uniquement en boucle locale et refuserait donc toute requête venant de l'extérieur.

Le numéro de série de la zone est figé. Avec une valeur liée au temps, le fichier changerait à chaque exécution, le déclencheur se lancerait à chaque passage et l'exigence de reproductibilité serait perdue.

Les requêtes sortantes sont transmises à un résolveur, faute de quoi la machine ne pourrait résoudre aucun nom externe.

### 5.4 Serveur de fichiers

Le service repose sur SSH, déjà présent, auquel on ajoute une configuration dédiée dans le répertoire d'inclusion de la distribution plutôt qu'en modifiant le fichier fourni par le système.

Un compte de service dédié est créé, sans shell de connexion, avec un espace cloisonné. Le cloisonnement ne cible que le groupe de ce compte : le compte d'administration conserve un accès complet, sans quoi on se priverait soi-même de la main sur la machine.

**Point de vigilance : validation avant écriture.**

La configuration SSH est validée par le démon lui-même avant d'être écrite à sa place définitive. Si la syntaxe est refusée, le fichier n'est pas déployé et le service n'est pas redémarré.

Cette précaution n'est pas théorique. Une erreur de configuration du service d'accès distant se paie immédiatement : on perd la main sur la machine et il ne reste que la console de l'hyperviseur pour réparer.

### 5.5 Base de données

MariaDB est installé avec le pilote Python qui permet aux modules Ansible de dialoguer avec le serveur. Sans ce pilote, la création de la base échoue sur une erreur de dépendance difficile à interpréter.

La base et le compte applicatif sont créés, avec les droits restreints à cette base. Le mot de passe n'est défini qu'à la création du compte : le rejouer à chaque exécution provoquerait un changement permanent.

### 5.6 Supervision

Un exportateur de métriques est installé et activé. Le service expose ses mesures sur le port prévu par le projet.

Une tâche d'attente vérifie que le port répond réellement avant de clore le rôle. Un service déclaré démarré signifie que le système a lancé le processus, pas que l'application écoute. Cette vérification évite de découvrir une panne trois étapes plus loin.

## 6. Variables, gabarits et déclencheurs

Les trois exigences de standardisation sont couvertes de la façon suivante.

**Les variables** sont rassemblées dans un fichier unique, organisé par rôle concerné. Aucune valeur n'est écrite en dur dans les tâches. Les variables de connexion restent dans l'inventaire : les dupliquer créerait deux sources de vérité qui finiraient par diverger.

| Variable | Emploi |
|---|---|
| `site_name` | nom du site affiché sur la page web |
| `site_root` | répertoire servi par Apache |
| `web_port` | port d'écoute du service web |
| `dns_domain` | zone DNS gérée |
| `dns_record`, `dns_record_ip` | enregistrement publié vers le serveur web |
| `dns_forwarder` | résolveur utilisé pour les noms externes |
| `sftp_user`, `sftp_group`, `sftp_dir` | compte, groupe et racine du service de fichiers |
| `db_name`, `db_user`, `db_password` | base et compte applicatif |
| `monitoring_port` | port d'exposition des métriques |
| `common_packages`, `common_timezone` | socle commun |

**Les gabarits** sont au nombre de six : le fichier de correspondance noms et adresses du socle, la page web, les trois fichiers de configuration du service DNS, et la configuration du service de fichiers. Chacun produit un contenu déterministe : aucune date, aucun identifiant aléatoire, aucune valeur qui change d'une exécution à l'autre.

**Les déclencheurs** sont au nombre de trois : redémarrage d'Apache, redémarrage de Bind9 et redémarrage du service SSH. Chacun est associé aux gabarits qui le concernent.

Un point de mécanique mérite d'être noté, parce qu'il se voit dans les relevés. Trois gabarits du rôle DNS changent au premier passage, donc trois notifications sont envoyées, et le déclencheur ne s'exécute **qu'une seule fois**, en fin de jeu. Ansible déduplique les demandes. Sans ce comportement, le service redémarrerait trois fois pour rien.

## 7. Preuve d'idempotence

L'exigence de recette est la suivante : la seconde exécution ne doit provoquer aucun changement inutile.

Le protocole retenu est celui qui a la plus grande valeur de démonstration. Les cinq machines ont été **arrêtées, détruites, puis recréées** à partir du modèle décrit dans le document précédent. L'infrastructure part donc d'un état vierge, sans aucune trace d'une exécution antérieure.

Le projet est ensuite exécuté deux fois de suite.

### 7.1 Première exécution

| Machine | Tâches réussies | Modifications | Échecs | Injoignables |
|---|---|---|---|---|
| web01 | 11 | **6** | 0 | 0 |
| dns01 | 13 | **8** | 0 | 0 |
| srv01 | 15 | **9** | 0 | 0 |
| db01 | 11 | **6** | 0 | 0 |
| mon01 | 10 | **4** | 0 | 0 |

L'infrastructure complète est construite depuis un état vide, sur les cinq machines, sans intervention manuelle.

### 7.2 Seconde exécution

| Machine | Tâches réussies | Modifications | Échecs | Injoignables |
|---|---|---|---|---|
| web01 | 10 | **0** | 0 | 0 |
| dns01 | 12 | **0** | 0 | 0 |
| srv01 | 14 | **0** | 0 | 0 |
| db01 | 11 | **0** | 0 | 0 |
| mon01 | 10 | **0** | 0 | 0 |

Aucune modification. L'état décrit dans le dépôt correspond exactement à l'état des machines.

### 7.3 Lecture des relevés

Le comptage des tâches réussies entre les deux exécutions porte une information intéressante, et elle confirme le bon fonctionnement des déclencheurs.

| Machine | 1re exécution | 2e exécution | Écart | Déclencheur |
|---|---|---|---|---|
| db01 | 11 | 11 | 0 | aucun |
| mon01 | 10 | 10 | 0 | aucun |
| web01 | 11 | 10 | 1 | oui |
| dns01 | 13 | 12 | 1 | oui |
| srv01 | 15 | 14 | 1 | oui |

L'écart vaut exactement un sur les trois machines dont un rôle porte un déclencheur, et zéro sur les deux autres.

Autrement dit : au premier passage, le gabarit a changé, donc le service a été redémarré, donc une tâche supplémentaire a été comptée. Au second passage, le gabarit était identique, donc aucune notification n'a été envoyée, donc aucune tâche supplémentaire.

C'est la démonstration que les déclencheurs sont bien conditionnés par un changement réel, et non exécutés systématiquement.

## 8. Vérification des services

Chaque service est vérifié par une commande, conformément au cahier des charges.

| Machine | Service | Vérification | Résultat attendu |
|---|---|---|---|
| WEB01 | Apache | requête sur le port 80 | page du site servie |
| DNS01 | Bind9 | interrogation de la zone locale | réponse faisant autorité |
| SRV01 | SFTP | ouverture d'une session | session établie, répertoire personnel |
| DB01 | MariaDB | état du service | service actif |
| MON01 | exportateur | ports en écoute et réponse | port exposé, service actif |

Les relevés obtenus sont les suivants.

**Serveur web.** La requête renvoie la page attendue, avec le nom du site et l'adresse de la machine qui a répondu. Le service est fonctionnel depuis l'extérieur de la machine.

**Serveur DNS.** L'interrogation de la zone locale renvoie un statut sans erreur, avec le drapeau *authoritative*, ce qui confirme que la réponse vient bien de la zone du projet et non d'un résolveur intermédiaire. L'enregistrement publié pointe vers l'adresse du serveur web.

**Serveur de fichiers.** Une session est établie avec le compte d'administration, sans mot de passe, et le répertoire courant correspond à son répertoire personnel. Le cloisonnement du compte de service, lui, ne concerne pas ce compte.

**Base de données.** Le service est actif et à l'écoute sur son port. La base et le compte applicatif ont été créés par le rôle.

**Supervision.** Le port dédié est en écoute et le service répond. Le relevé des ports confirme l'exposition attendue.

## 9. Points de vigilance récapitulés

| Constat | Conséquence | Mesure appliquée |
|---|---|---|
| Le poste d'administration n'a pas de route vers le réseau des machines | Toutes les connexions restent en attente | Option de saut déclarée dans l'inventaire |
| Les empreintes SSH changent après un reclonage | Connexion refusée sur conflit d'empreinte | Acceptation des clés inconnues, purge préalable |
| Une erreur de configuration SSH coupe l'accès distant | Perte de la main sur la machine | Validation de la syntaxe avant écriture |
| Le service DNS n'écoute qu'en boucle locale par défaut | Aucune réponse aux requêtes externes | Écoute explicitement étendue à l'adresse de la machine |
| Un numéro de série de zone variable change à chaque passage | Déclencheur relancé et reproductibilité perdue | Numéro de série figé |
| Un mot de passe rejoué à chaque exécution produit un changement | Second passage non conforme | Mot de passe posé uniquement à la création |
| Les modules de base de données exigent un pilote Python | Échec sur erreur de dépendance | Pilote installé avec le serveur |
| Un service déclaré démarré n'écoute pas forcément | Panne découverte trop tard | Attente active de l'écoute réseau |
| Le rafraîchissement de la liste des paquets est non borné | Changement à chaque exécution | Rafraîchissement borné dans le temps |
| Le service SSH est réparti sur plusieurs fichiers de configuration | Fichier fourni par la distribution écrasé | Configuration ajoutée dans le répertoire d'inclusion |

## Annexe A — Séquence de déploiement

```bash
# Vérification de l'inventaire
ansible-inventory -i inventory/hosts.ini --graph
ansible -i inventory/hosts.ini all -m ping

# Contrôle de syntaxe sans exécution
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --list-tasks

# Développement d'un rôle, sur une seule machine
ansible-playbook site.yml --limit web01

# Première exécution complète
ansible-playbook -i inventory/hosts.ini site.yml

# Seconde exécution immédiate : contrôle de reproductibilité
ansible-playbook -i inventory/hosts.ini site.yml
```

Commandes de vérification des services.

```bash
ansible web01 -a "curl -s -o /dev/null -w %{http_code} http://127.0.0.1"
ansible dns01 -a "dig @192.168.10.21 www.netsys.test"
sftp -o ProxyJump=root@192.168.1.150 admin@192.168.10.22
ansible db01 -a "systemctl status mariadb --no-pager"
ansible mon01 -a "ss -lntup"
```

## Annexe B — Ce que la démarche démontre

La valeur de ce travail ne tient pas au fait que les cinq services fonctionnent. Elle tient au fait qu'ils peuvent être **détruits et reconstruits**.

Les cinq machines ont été supprimées, puis recréées depuis un modèle, puis configurées par une seule commande. Aucune configuration n'a été retapée, aucune mémoire n'a été sollicitée.

C'est aussi ce qui rend le diagnostic possible. Quand une dérive est introduite sur une machine, il n'y a pas à deviner ce qu'elle contenait : le dépôt décrit l'état cible, la commande le rétablit, et la relecture des relevés confirme que le retour à l'état attendu a bien eu lieu.

La documentation du socle et celle du déploiement forment une chaîne complète : une machine préparée une fois, clonée en cinq exemplaires, puis configurée par le code. Reconstruire l'infrastructure à l'identique revient à rejouer la séquence, pas à reproduire un savoir-faire.
