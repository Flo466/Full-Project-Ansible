# Création reproductible de machines virtuelles Debian sur Proxmox VE

> **Contexte**
>
> NETSYS doit pouvoir reconstruire son infrastructure à partir de machines Debian remises dans un état propre. Le déploiement doit être reproductible, documenté et vérifiable.
>
> Pour répondre à cette exigence, l'infrastructure repose sur cinq machines virtuelles Debian 13, chacune portant un service distinct : serveur web, serveur DNS, serveur SFTP, base de données MariaDB et supervision.
>
> Créer ces cinq machines à la main, une par une, depuis un installeur classique, revient à répéter cinq fois les mêmes vingt minutes d'installation, avec cinq occasions de divergence. La méthode retenue supprime ce problème : on prépare une image système unique et propre, on la fige sous forme de modèle, puis on en dérive les cinq machines par clonage.
>
> Toute la configuration qui différencie une machine d'une autre, nom d'hôte, adresse IP, compte d'administration, clé publique SSH, est injectée automatiquement au premier démarrage par cloud-init. Aucune intervention manuelle sur l'installeur n'est nécessaire.

- **Date** : 7 octobre 2026
- **Hyperviseur** : Proxmox VE 8.4, nœud unique
- **Modèle** : Debian 13 (Trixie) generic cloud
- **Machines produites** : 5
- **Mode de clonage** : clones liés sur disque QCOW2

---

## 1. Objectif et périmètre

Le présent document décrit la chaîne complète de provisionnement, depuis le téléchargement de l'image système jusqu'à l'obtention des cinq machines adressées et joignables.

Ce document **ne couvre pas** le déploiement des services applicatifs. Ceux-ci sont installés et configurés par Ansible, depuis les rôles du dépôt de projet. Toute modification manuelle d'un service dans une machine ne constitue pas une validation.

Le périmètre s'arrête donc à l'obtention d'un socle homogène : cinq machines identiques, propres, adressées, accessibles par clé SSH et pilotables par l'API Proxmox.

## 2. Socle technique

### 2.1 Stockages

Trois stockages sont déclarés sur l'hyperviseur.

| Identifiant | Type | Chemin | Contenu activé | Emploi |
|---|---|---|---|---|
| `local` | dir | `/var/lib/vz` | backup, iso, vztmpl | ISO et sauvegardes |
| `local-lvm` | lvmthin | volume group `pve` | images, rootdir | machines d'autres chantiers |
| `ssd-pcie-externe` | dir | `/mnt/vms-data` | iso, images | **terrain du présent projet** |

Toutes les images de machines du projet sont stockées sur `ssd-pcie-externe`, soit 245 Go d'espace dont l'essentiel est libre.

Point de vocabulaire souvent source de confusion : ce que l'on appelle communément « pool » recouvre trois objets sans rapport. Un **zpool** est un agrégat ZFS au niveau noyau, et il n'en existe aucun sur cet hyperviseur. Un **stockage Proxmox** est une entrée de `storage.cfg`, c'est lui qui apparaît avant les deux points dans une référence de volume. Un **resource pool** est un simple regroupement logique de machines, sans lien avec le stockage. Le nom `ssd-pcie-externe` désigne un stockage de type `dir`, c'est-à-dire un dossier sur un système de fichiers ext4, et non un agrégat ZFS.

### 2.2 Réseau

Les machines du projet sont rattachées à un pont interne dédié, `vmbr1`, porté par l'hyperviseur.

| Interface | Adressage | Rôle |
|---|---|---|
| `vmbr0` | 192.168.1.150/24 | réseau local, accès à l'interface d'administration |
| `vmbr1` | 192.168.10.1/24 | réseau du projet, traduction d'adresse vers `vmbr0` |

Le pont `vmbr1` est déclaré sans port physique, avec une règle de traduction d'adresse source vers `vmbr0`. Les machines y trouvent leur passerelle en 192.168.10.1 et atteignent l'extérieur par cette traduction.

**Chaque machine ne porte qu'une seule interface réseau.** Ajouter une seconde carte avec une passerelle supplémentaire introduirait une seconde route par défaut et provoquerait un routage asymétrique, donc des connexions qui s'établissent dans un sens et se perdent dans l'autre. La sortie vers l'extérieur est déjà assurée par la traduction d'adresse, elle ne justifie pas une interface supplémentaire.

### 2.3 Identification des machines

| Plage | Emploi |
|---|---|
| 9001 | modèle de projet |
| 400 à 404 | machines du projet |

Les plages 2xx et 3xx sont occupées par d'autres chantiers et ne sont pas touchées.

## 3. Préparation de l'image système

### 3.1 Installeur ou image cloud

Une image d'installation classique n'est pas utilisable telle quelle. Elle contient un programme d'installation, pas un système installé. La dériver en modèle imposerait de traverser cinq à six écrans d'installation, puis de recommencer pour chaque machine.

L'image cloud répond exactement au besoin inverse : c'est un **disque système déjà installé**, livré comprimé sous forme de fichier QCOW2, qui ne demande qu'à être branché. Debian ne fournit ni cloud-init ni compte utilisateur, mais l'amorçage est fonctionnel et prend en charge l'injection de configuration.

L'image est téléchargée depuis le dépôt officiel, accompagnée de son empreinte SHA-512.

```bash
cd /root
curl -fLO https://cloud.debian.org/images/cloud/trixie/latest/debian-13-genericcloud-amd64.qcow2
curl -fLO https://cloud.debian.org/images/cloud/trixie/latest/SHA512SUMS
sha512sum -c --ignore-missing SHA512SUMS
```

L'option `-f` fait échouer la commande en cas d'erreur serveur plutôt que de produire un fichier tronqué silencieusement. La vérification de l'empreinte doit afficher `OK` avant de poursuivre.

### 3.2 Point de vigilance : lenteur de l'amorçage réseau

La commande peut rester plusieurs minutes sans recevoir un seul octet avant de démarrer brutalement à pleine vitesse, de l'ordre de 45 Mo/s. Ce comportement traduit une négociation réseau longue, généralement liée à la résolution de nom ou à une route IPv6 non aboutie.

La mesure à en tirer est de ne pas interrompre une commande trop tôt sur un simple affichage figé. En cas de blocage confirmé, forcer la pile IPv4 avec `curl -4` et reprendre le transfert avec `-C -`.

## 4. Création de la machine modèle

### 4.1 Déclaration de la machine

```bash
qm create 9001 --name tpl-debian13 \
  --memory 1024 --cores 2 --ostype l26 \
  --scsihw virtio-scsi-single \
  --net0 virtio,bridge=vmbr1 \
  --agent enabled=1 \
  --pool TP-Ansible
```

Le contrôleur de disque retenu est `virtio-scsi-single`. Le contrôleur générique `virtio-scsi-pci` déclenche au démarrage un avertissement indiquant que l'option `iothread` n'est pas valide sur ce type de contrôleur. Cet avertissement n'empêche pas le fonctionnement, mais il est hérité par chaque clone et finit par saturer les journaux.

L'option `--agent enabled=1` branche le canal de communication invité vers l'hôte. Elle ne s'occupe que du canal, pas du logiciel qui doit répondre à l'autre bout.

### 4.2 Import du disque

```bash
qm importdisk 9001 /root/debian-13-genericcloud-amd64.qcow2 ssd-pcie-externe --format qcow2
```

**Le format `qcow2` est obligatoire dans le cadre de ce projet.** Sans l'option explicite, Proxmox convertit l'image au format `raw`, et un disque `raw` ne peut pas servir de base à un clonage lié. Proxmox refuse alors l'opération avec un message indiquant que le clonage lié n'est pas supporté par ce format.

Ce point est structurant : c'est le format QCOW2 qui autorise la superposition d'un fichier de différences au-dessus d'une base commune, et donc l'existence même des clones liés.

La commande retourne un volume déclaré comme `unused0`. Il faut lire la chaîne exacte retournée avant de poursuivre, soit sous la forme `ssd-pcie-externe:9001/vm-9001-disk-0.qcow2`.

### 4.3 Finalisation du disque et des lecteurs

```bash
qm set 9001 --scsi0 ssd-pcie-externe:9001/vm-9001-disk-0.qcow2,discard=on
qm set 9001 --ide2 ssd-pcie-externe:cloudinit
qm set 9001 --boot order=scsi0
qm resize 9001 scsi0 10G
```

Le paramètre `discard=on` autorise la machine à signaler au stockage les blocs qu'elle n'utilise plus. C'est utile sur une image système de petite taille étendue à 10 Go.

Le lecteur cloud-init est généré automatiquement par la seconde commande. Proxmox produit un petit disque de quelques mégaoctets, présenté comme un lecteur optique, qui contient la configuration à appliquer au premier démarrage.

Le lecteur cloud-init n'est pas écrit avec `--cicustom`. Cette variante aurait exigé de déposer un fichier YAML dans un stockage déclaré avec le contenu `snippets`, or aucun des stockages de cet hyperviseur n'a ce contenu activé. Le lecteur cloud-init standard couvre le besoin sans dépendance supplémentaire.

L'ordre d'amorçage doit être explicitement fixé sur le disque système. À défaut, la machine tente un démarrage réseau et part en attente sur une tentative PXE.

## 5. Injection de la configuration par cloud-init

```bash
qm set 9001 --ciuser admin --sshkeys /root/cle-admin.pub
qm set 9001 --ipconfig0 ip=192.168.10.10/24,gw=192.168.10.1 --nameserver 192.168.1.1
```

Trois éléments sont posés ici.

Le compte d'administration `admin` est créé automatiquement au premier démarrage. La clé publique SSH fournie est déposée dans son répertoire d'autorisation, ce qui dispense de tout mot de passe. L'adressage réseau complet, adresse, passerelle et résolveur, est appliqué à l'interface.

Deux remarques d'exploitation.

Le résolveur de noms doit être fourni explicitement. Le pont interne n'expose aucun service de résolution, donc sans cette indication les machines démarrent sans DNS opérationnel et tout `apt update` échoue.

L'adresse posée dans le modèle est **volontairement hors de la plage des machines finales**, ici 192.168.10.10, alors que les machines produites occuperont 192.168.10.20 à 192.168.10.24. L'adresse d'un modèle n'a pas vocation à être conservée, puisqu'elle sera remplacée sur chaque clone. La placer hors plage évite qu'un modèle démarré par erreur ne vienne en conflit avec une machine en service.

## 6. Premier démarrage et personnalisation de l'invité

Une seule chose doit être ajoutée dans le système invité : l'agent de communication invité.

```bash
qm start 9001
# puis, dans la machine
sudo apt update && sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

Cet agent est le correspondant du canal ouvert par `--agent enabled=1`. Sans lui, l'hyperviseur ne peut ni lire l'adressage de la machine, ni demander un arrêt propre, ni geler le système de fichiers avant un instantané. Le canal serait établi, mais muet.

L'image cloud Debian ne l'embarque pas par défaut, ce qui explique qu'il faille l'installer explicitement. L'installer sur le modèle plutôt que sur chaque machine produite est un gain direct : cinq machines en hériteront sans qu'une seule commande soit répétée.

Contrôle depuis l'hyperviseur :

```bash
qm guest cmd 9001 network-get-interfaces
```

L'affichage de l'adresse attendue confirme que le canal fonctionne dans les deux sens.

## 7. Remise à l'état propre

Cette étape est **indispensable avant la conversion en modèle**. Elle est également la plus facile à oublier.

```bash
sudo cloud-init clean --logs --machine-id
sudo rm -f /etc/ssh/ssh_host_*
sudo truncate -s 0 /etc/machine-id
sudo systemctl poweroff
```

Trois éléments sont remis à zéro, chacun pour une raison précise.

**L'identifiant machine** identifie un système de façon unique sur le réseau. Deux machines le partageant se comportent comme une seule du point de vue de plusieurs services.

**Les clés d'hôte SSH** constituent l'identité réseau d'une machine. Si cinq clones partagent les mêmes, tout client légitime reçoit une alerte de changement d'empreinte à chaque connexion, et la seule manière de la faire disparaître est de désactiver la vérification, ce qui revient à retirer la protection.

**L'historique cloud-init** est purgé pour que l'injection se rejoue intégralement au premier démarrage de chaque clone.

Proxmox régénère les clés d'hôte absentes et réinstalle un identifiant machine unique au redémarrage, sous réserve que cloud-init soit autorisé à se réexécuter, ce qui est le cas après `clean --machine-id`.

### Point de vigilance : purge et régénération des clés d'hôte

Un piège classique consiste à lancer la régénération sans avoir supprimé les clés existantes. L'outil de génération de clés d'hôte **ne remplace jamais une clé déjà présente**, il ne crée que les clés manquantes, et il le fait sans afficher le moindre message. La commande se termine avec succès et le nœud conserve donc l'identité de sa source.

La vérification se fait en lisant le commentaire des clés publiques : si le commentaire désigne un autre nom de machine que celle sur laquelle on se trouve, le partage d'identité est établi.

La règle opérationnelle est de toujours supprimer avant de régénérer, et de ne jamais appliquer cette opération à la machine source, où elle détruirait l'accès SSH en cours.

## 8. Conversion en modèle

```bash
qm status 9001
qm template 9001
```

La machine doit être arrêtée. La conversion la fait passer en lecture seule et marque son disque de base comme protégé : Proxmox pose un attribut d'immuabilité sur le fichier de base, précisément pour empêcher qu'une suppression accidentelle ne corrompe les machines qui en dépendent.

## 9. Production des cinq machines

```bash
for i in 0 1 2 3 4; do
  qm clone 9001 $((400+i)) \
    --name $(echo web01 dns01 srv01 db01 mon01 | cut -d' ' -f$((i+1))) \
    --full 0 --pool TP-Ansible
done
```

Le paramètre `--full 0` demande un **clonage lié**. Chaque clone ne stocke que ses différences par rapport au disque du modèle, ce qui ramène l'empreinte de chaque machine à quelques mégaoctets au lieu de plusieurs gigaoctets.

Deux comportements observés à la création méritent d'être connus.

Le disque du modèle est renommé en `base-9001-disk-0.qcow2` et devient la base commune, référencée comme fichier parent par chacun des clones.

Le lecteur cloud-init, lui, est cloné en **complet**. Ce lecteur n'a pas de base partagée, il est donc dupliqué pour chaque machine. C'est une bonne nouvelle : chaque clone dispose de son propre fichier de configuration, et les adresses peuvent donc diverger d'une machine à l'autre.

## 10. Adressage des machines produites

Les clones héritent de la configuration cloud-init du modèle, **y compris de son adresse**. Cinq machines démarrées dans cet état se disputeraient la même adresse. L'adressage individuel doit donc être appliqué **avant le premier démarrage**.

| Machine | Identifiant | Adresse | Service |
|---|---|---|---|
| web01 | 400 | 192.168.10.20 | Serveur web |
| dns01 | 401 | 192.168.10.21 | Serveur DNS |
| srv01 | 402 | 192.168.10.22 | Serveur SFTP |
| db01 | 403 | 192.168.10.23 | Base de données MariaDB |
| mon01 | 404 | 192.168.10.24 | Supervision |

```bash
for spec in "400 20" "401 21" "402 22" "403 23" "404 24"; do
  set -- $spec
  qm set $1 --ipconfig0 ip=192.168.10.$2/24,gw=192.168.10.1 --nameserver 192.168.1.1
done
```

Le résolveur est repris à chaque appel car la commande **remplace** l'intégralité de la ligne de configuration réseau, elle ne fusionne pas avec l'existant.

## 11. Vérifications

### 11.1 Connectivité

Depuis l'hyperviseur, qui porte l'adresse de passerelle du réseau interne :

```bash
for id in 400 401 402 403 404; do qm start $id; done
sleep 75
for ip in 20 21 22 23 24; do
  ping -c1 -W2 192.168.10.$ip >/dev/null && echo "OK 192.168.10.$ip" || echo "KO 192.168.10.$ip"
done
```

Le délai d'attente couvre la réexécution complète de cloud-init, qui crée le compte, dépose la clé, applique l'adressage et régénère les clés d'hôte.

### 11.2 Accès depuis le poste d'administration

Le poste d'administration se trouve sur le réseau local, dans le segment 192.168.1.0/24, et n'a aucune route vers le réseau interne des machines. L'accès passe donc par l'hyperviseur, qui voit les deux réseaux.

```bash
ssh -J root@192.168.1.150 admin@192.168.10.20
```

Cette contrainte se transpose dans la configuration d'Ansible, par une option de saut appliquée à tous les hôtes.

### 11.3 Point de vigilance : empreintes SSH en conflit

À la première connexion vers une machine produite, un client SSH peut refuser la connexion en signalant un changement d'empreinte. La cause est une entrée périmée dans le fichier `known_hosts`, laissée par une machine antérieure portant la même adresse.

Le point à retenir est que l'option d'acceptation automatique des clés inconnues **ne concerne que les clés inconnues**. Une clé en conflit avec une entrée existante est refusée quelle que soit l'option. Seule la purge de l'entrée résout le problème.

```bash
ssh-keygen -R 192.168.10.20
```

### 11.4 Nom d'hôte

Proxmox ne transmet pas le nom de la machine au système invité. Les cinq clones portent donc encore le nom du modèle dans leur système d'exploitation.

Ce point n'est pas un défaut de la chaîne de provisionnement, c'est une limite de l'injection cloud-init dans cette configuration. Il est traité par l'outillage de configuration, dont le rôle commun pose le nom d'hôte et régénère le fichier de correspondance noms et adresses.

## 12. Points de vigilance récapitulés

| Constat | Conséquence | Mesure appliquée |
|---|---|---|
| L'import de disque produit du `raw` par défaut | Le clonage lié est refusé | Import explicite au format `qcow2` |
| Une image de base est verrouillée en immuable | La suppression du modèle échoue | Modèle conservé comme base, jamais modifié |
| Les clones héritent de l'adressage du modèle | Conflit d'adresses au démarrage | Adressage individuel appliqué avant démarrage |
| Le résolveur n'est pas fourni par le pont interne | Absence de DNS, mises à jour impossibles | Résolveur explicitement injecté |
| La régénération de clés ne remplace pas l'existant | Identité partagée entre machines | Suppression préalable systématique |
| L'option d'acceptation SSH ne tolère pas un conflit | Connexion refusée | Purge de l'empreinte périmée |
| Le contrôleur `virtio-scsi-pci` rejette `iothread` | Avertissement hérité par chaque clone | Contrôleur `virtio-scsi-single` |
| Aucun stockage n'a le contenu `snippets` | Inutilisable pour un fichier cloud-init externe | Lecteur cloud-init standard |
| Proxmox ne transmet pas le nom de la machine | Clones nommés comme le modèle | Nom d'hôte posé par l'outillage de configuration |

## Annexe A — Séquence complète

```bash
# 1. Récupération de l'image système
cd /root
curl -fLO https://cloud.debian.org/images/cloud/trixie/latest/debian-13-genericcloud-amd64.qcow2
curl -fLO https://cloud.debian.org/images/cloud/trixie/latest/SHA512SUMS
sha512sum -c --ignore-missing SHA512SUMS

# 2. Déclaration de la machine modèle
qm create 9001 --name tpl-debian13 --memory 1024 --cores 2 --ostype l26 \
  --scsihw virtio-scsi-single --net0 virtio,bridge=vmbr1 \
  --agent enabled=1 --pool TP-Ansible

# 3. Import du disque système au format qcow2
qm importdisk 9001 /root/debian-13-genericcloud-amd64.qcow2 ssd-pcie-externe --format qcow2

# 4. Disque, lecteur cloud-init, amorçage, extension
qm set 9001 --scsi0 ssd-pcie-externe:9001/vm-9001-disk-0.qcow2,discard=on
qm set 9001 --ide2 ssd-pcie-externe:cloudinit
qm set 9001 --boot order=scsi0
qm resize 9001 scsi0 10G

# 5. Configuration injectée au premier démarrage
qm set 9001 --ciuser admin --sshkeys /root/cle-admin.pub
qm set 9001 --ipconfig0 ip=192.168.10.10/24,gw=192.168.10.1 --nameserver 192.168.1.1

# 6. Premier démarrage, puis dans l'invité : agent de communication
qm start 9001
# sudo apt update && sudo apt install -y qemu-guest-agent
# sudo systemctl enable --now qemu-guest-agent

# 7. Remise à l'état propre
# sudo cloud-init clean --logs --machine-id
# sudo rm -f /etc/ssh/ssh_host_*
# sudo truncate -s 0 /etc/machine-id
# sudo systemctl poweroff

# 8. Conversion en modèle
qm template 9001

# 9. Production des cinq machines
for i in 0 1 2 3 4; do
  qm clone 9001 $((400+i)) \
    --name $(echo web01 dns01 srv01 db01 mon01 | cut -d' ' -f$((i+1))) \
    --full 0 --pool TP-Ansible
done

# 10. Adressage individuel
for spec in "400 20" "401 21" "402 22" "403 23" "404 24"; do
  set -- $spec
  qm set $1 --ipconfig0 ip=192.168.10.$2/24,gw=192.168.10.1 --nameserver 192.168.1.1
done

# 11. Démarrage et contrôle
for id in 400 401 402 403 404; do qm start $id; done
for ip in 20 21 22 23 24; do
  ping -c1 -W2 192.168.10.$ip >/dev/null && echo "OK 192.168.10.$ip" || echo "KO 192.168.10.$ip"
done
```

## Annexe B — Ce que la chaîne apporte

La chaîne décrite produit cinq machines en quelques minutes, là où cinq installations manuelles demanderaient plusieurs heures.

Elle garantit surtout que les cinq machines sont **rigoureusement identiques** sur toute la partie qui ne doit pas varier : noyau, paquets de base, agent de communication, état de cloud-init. Seule la configuration destinée à différer diffère, et elle est injectée, pas saisie.

C'est cette séparation entre un socle figé et une configuration injectée qui rend le déploiement reproductible. Reconstruire l'infrastructure à l'identique revient à rejouer la séquence, pas à reproduire un savoir-faire manuel.
