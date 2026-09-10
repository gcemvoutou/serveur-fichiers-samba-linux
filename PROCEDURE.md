# Procédure : Serveur de fichiers Samba (Debian 13 / VirtualBox)

> Ce document détaille chaque étape réalisée, avec la commande, sa justification technique, et le résultat attendu. Les captures d'écran sont référencées à l'endroit où elles doivent être insérées (dossier `images/`).

## Sommaire

1. [Prérequis](#1-prérequis)
2. [Architecture](#2-architecture)
3. [Création des machines virtuelles](#3-création-des-machines-virtuelles)
4. [Installation de Debian](#4-installation-de-debian-sur-les-deux-vms)
5. [Configuration réseau (nmcli)](#5-configuration-réseau-nmcli)
6. [Samba](#6-samba-uniquement-sur-srvfichiers)
7. [Quotas disque](#7-quotas-disque-srvfichiers)
8. [Sauvegarde automatisée](#8-sauvegarde-automatisée-srvfichiers--srvsauvegarde)
9. [Pare-feu UFW](#9-pare-feu-ufw-srvfichiers)
10. [Tests finaux](#10-tests-finaux)
11. [Pistes d'amélioration](#11-pistes-damélioration)

---

## 1. Prérequis

- [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
- Image [Debian 13 "Trixie" netinst (amd64)](https://www.debian.org/distrib/)

> [!TIP]
> Une seule image `.iso` sert pour les deux VMs (SrvFichiers et SrvSauvegarde).

---

## 2. Architecture

Deux VMs, chacune avec deux cartes réseau : une en **NAT** (accès Internet), une en **Host-only** (communication entre les VMs et avec le poste client).

| Machine | Rôle | RAM / vCPU | Disque | IP Host-only |
|---|---|---|---|---|
| **SrvFichiers** | Serveur Samba principal | 2048 Mo / 2 vCPU | 20 Go dynamique | 192.168.56.10 |
| **SrvSauvegarde** | Réception des sauvegardes | 512 Mo / 1 vCPU | 15 Go dynamique | 192.168.56.20 |
| PC hôte | Poste client de test | — | — | 192.168.56.12 (auto) |

> [!NOTE]
> Le réseau **Host-only** est préféré au « Réseau interne » : il permet la communication entre les deux VMs **et** l'accès depuis le PC hôte, indispensable pour tester le partage Samba depuis l'explorateur Windows.

---

## 3. Création des machines virtuelles

Pour chaque VM : Nouvelle machine → Type Linux → Version Debian (64-bit) → RAM/vCPU/disque selon le tableau ci-dessus → Stockage : monter l'ISO Debian 13 → Réseau : Adaptateur 1 = NAT, Adaptateur 2 = Host-only (`vboxnet0`).

> [!WARNING]
> Si « Host-only » n'apparaît pas dans la liste des adaptateurs : VirtualBox → Outils → Réseau → Réseaux uniquement-hôte → Créer.

📷 **`images/01-vm-srvfichiers-parametres.png`** Paramètres de la VM SrvFichiers (mémoire, disque, réseau).

---

## 4. Installation de Debian (sur les deux VMs)

### 4.1 Langue, clavier, identité
Langue française, clavier France. Nom d'hôte : `SrvFichiers` ou `SrvSauvegarde`. Mot de passe root + utilisateur standard.

### 4.2 Partitionnement
« Assisté – disque entier » → « Tout dans une seule partition ».

> Les quotas (partie 7) s'appliquent par partition une seule partition = un seul point à gérer.

### 4.3 Sélection des paquets (tasksel)
Tenter de décocher « Environnement de bureau » et « GNOME », garder « Utilitaires usuels » + « Serveur SSH ». Installer GRUB sur `/dev/sda`.

> [!WARNING]
> Si le bureau s'installe quand même (image utilisée, case mal cochée...), ce n'est pas grave → voir 4.5.

### 4.4 Clavier en AZERTY

```bash
dpkg-reconfigure keyboard-configuration
```
Dans les menus : **Generic 105-key PC → French → French → The default for the keyboard layout → No compose key → No**

```bash
setupcon   # applique sans redémarrer
```

### 4.5 Retirer l'environnement de bureau (si nécessaire)

```bash
systemctl get-default
# graphical.target = le système démarre sur l'interface graphique
```

```bash
su -                                      # Debian ne met pas l'utilisateur dans sudo par défaut
systemctl set-default multi-user.target   # bascule en mode texte
reboot
```

📷 **`images/02-installation-mode-texte.png`**  Invite de connexion en mode texte après redémarrage.

```bash
ls /usr/share/xsessions/     # confirme l'environnement présent (gnome.desktop)
apt purge gnome-core gnome-shell gdm -y
apt autoremove --purge -y
```

📷 **`images/03-purge-gnome.png`**  Résultat de `apt purge gnome-core gnome-shell gdm -y`.

> `purge` supprime aussi les fichiers de configuration (contrairement à `remove`), pour un nettoyage complet.

```bash
systemctl get-default   # doit maintenant répondre : multi-user.target
```

---

## 5. Configuration réseau (nmcli)

> [!IMPORTANT]
> Sur une installation avec bureau, le réseau est géré par **NetworkManager**, pas par `/etc/network/interfaces`. On utilise donc `nmcli`.

### 5.1 Identifier les interfaces
```bash
nmcli device status
```
`enp0s3` = NAT, `enp0s8` = Host-only (noms indicatifs, peuvent varier).

### 5.2 Créer un profil dédié par carte

Le profil générique unique créé à l'installation (« Wired connection 1 ») a tendance à sauter d'une carte à l'autre. On le supprime et on recrée deux profils propres :

```bash
nmcli connection delete "Wired connection 1"

# Carte NAT (DHCP)
nmcli connection add type ethernet ifname enp0s3 con-name NAT-enp0s3 \
  ipv4.method auto connection.autoconnect yes

# Carte Host-only (IP fixe) adapter l'adresse selon la VM (.10 ou .20)
nmcli connection add type ethernet ifname enp0s8 con-name HostOnly-enp0s8 \
  ipv4.method manual ipv4.addresses 192.168.56.10/24 connection.autoconnect yes
```

### 5.3 Activer et vérifier

```bash
nmcli connection up NAT-enp0s3
nmcli connection up HostOnly-enp0s8
nmcli device status
ping -c 3 192.168.56.20   # test croisé entre les deux VMs
```

📷 **`images/04-nmcli-ping.png`**  `nmcli device status` + ping réussi entre les deux VMs.

### 5.4 Mettre à jour le système
```bash
apt update
apt upgrade -y
```

### 5.5 Vérifier et activer SSH

> [!NOTE]
> Cette étape se fait **après** la configuration réseau : `apt install openssh-server` nécessite un accès Internet.

```bash
systemctl status ssh
# si "Unit ssh.service could not be found" :
apt install openssh-server -y
systemctl enable ssh --now
```

📷 **`images/05-ssh-connexions.png`** Connexions SSH réussies depuis le PC hôte vers `srvfichiers@192.168.56.10` et `srvsauvegarde@192.168.56.20`.

---

## 6. Samba (uniquement sur SrvFichiers)

Samba sert de traducteur entre le système Linux (serveur) et les postes Windows (clients), via le protocole **SMB/CIFS** le langage réseau natif de Windows.

### 6.1 Installation
```bash
apt install samba samba-common-bin acl -y
```

### 6.2 Groupes métiers
```bash
groupadd direction
groupadd compta
groupadd technique
```
> La gestion des droits par groupe (plutôt que par utilisateur) centralise les accès : ajouter/retirer un employé d'un groupe suffit à mettre à jour ses permissions, sans toucher à la configuration Samba.

### 6.3 Utilisateurs de test
```bash
adduser --no-create-home --disabled-password udirection
usermod -aG direction udirection

adduser --no-create-home --disabled-password ucompta
usermod -aG compta ucompta

adduser --no-create-home --disabled-password utechnique
usermod -aG technique utechnique

smbpasswd -a udirection
smbpasswd -a ucompta
smbpasswd -a utechnique
```
- `--no-create-home` : ces comptes n'accèdent qu'à Samba, pas besoin de `/home/user`.
- `--disabled-password` : désactive la connexion Linux classique (local/SSH), seul l'accès réseau Samba est possible.

> [!WARNING]
> Le mot de passe Samba (`smbpasswd`) est distinct du mot de passe Linux de l'utilisateur.

### 6.4 Arborescence et droits
```bash
mkdir -p /srv/partages/{direction,compta,technique,commun}

chown root:direction /srv/partages/direction && chmod 2770 /srv/partages/direction
chown root:compta    /srv/partages/compta    && chmod 2770 /srv/partages/compta
chown root:technique /srv/partages/technique && chmod 2770 /srv/partages/technique
chown root:root      /srv/partages/commun    && chmod 2777 /srv/partages/commun
```
> Le `2` devant les permissions active le bit **SetGID** : tout nouveau fichier hérite du groupe du dossier, pas du groupe personnel du créateur condition nécessaire pour que le cloisonnement par service fonctionne dans la durée.

Vérification :
```bash
ls -l /srv/partages/
```

### 6.5 Configuration `smb.conf`
```bash
nano /etc/samba/smb.conf
```
```ini
[global]
   workgroup = MAIRIE
   server string = SrvFichiers - Serveur de fichiers TP
   security = user
   map to guest = never

[Direction]
   path = /srv/partages/direction
   valid users = @direction
   write list = @direction
   create mask = 0660
   directory mask = 2770
   browsable = yes

[Compta]
   path = /srv/partages/compta
   valid users = @compta
   write list = @compta
   create mask = 0660
   directory mask = 2770
   browsable = yes

[Technique]
   path = /srv/partages/technique
   valid users = @technique
   write list = @technique
   create mask = 0660
   directory mask = 2770
   browsable = yes

[Commun]
   path = /srv/partages/commun
   valid users = @direction, @compta, @technique
   write list = @direction, @compta, @technique
   create mask = 0666
   directory mask = 2777
   browsable = yes
```

```bash
testparm                       # vérifie la syntaxe AVANT de redémarrer
systemctl restart smbd nmbd
systemctl enable smbd nmbd
```

**Tests d'accès** (depuis l'explorateur Windows, poste client configuré en 192.168.56.12) :

📷 **`images/06-samba-acces-autorise.png`** — Connexion réussie à `\\192.168.56.10\Compta` avec `ucompta`.

📷 **`images/07-samba-acces-refuse.png`** — Connexion refusée sur `\\192.168.56.10\Compta` avec `udirection`.

---

## 7. Quotas disque (SrvFichiers)

Objectif : empêcher qu'un service ne sature le disque du serveur.

| Groupe | Soft (alerte) | Hard (blocage) | Valeur en blocs (soft / hard) |
|---|---|---|---|
| compta | 450 Mo | 500 Mo | 460800 / 512000 |
| direction | 450 Mo | 500 Mo | 460800 / 512000 |
| technique | 900 Mo | 1024 Mo | 921600 / 1048576 |

```bash
apt install quota -y
nano /etc/fstab
```
Ajouter `usrquota,grpquota` sur la ligne de la partition racine :
```
/dev/sda1   /   ext4   errors=remount-ro,usrquota,grpquota   0   1
```
```bash
mount -o remount /
quotacheck -cug /
quotaon -v /
```
```bash
setquota -g compta    460800 512000 0 0 /
setquota -g direction 460800 512000 0 0 /
setquota -g technique 921600 1048576 0 0 /
repquota -g /
```

📷 **`images/08-repquota.png`** — Résultat de `repquota -g /` confirmant les limites par groupe.

| Paramètre | Type de limite | Comportement |
|---|---|---|
| Soft | Souple (alerte) | Autorise l'écriture mais avertit (période de grâce de 7 jours) |
| Hard | Stricte (blocage) | Interdit toute nouvelle écriture dès le seuil atteint |
| `0 0` finaux | Aucune | Pas de restriction sur le nombre de fichiers (inodes) |

---

## 8. Sauvegarde automatisée (SrvFichiers → SrvSauvegarde)

### 8.1 Préparer SrvSauvegarde
```bash
apt install openssh-server rsync -y
mkdir -p /srv/sauvegardes
adduser --disabled-password backupuser
chown backupuser:backupuser /srv/sauvegardes
passwd backupuser   # temporaire, le temps de copier la clé SSH
```

> [!WARNING]
> `backup` est un compte **système** déjà présent par défaut sur Debian (UID 34, shell `/usr/sbin/nologin`). Créer un utilisateur `backup` échoue, et même débloqué, ce compte système refuse toute connexion SSH. On utilise donc un nom dédié : **`backupuser`**.

### 8.2 Clé SSH (sur SrvFichiers)
```bash
ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_backup
ssh-copy-id -i /root/.ssh/id_backup.pub backupuser@192.168.56.20
```

📷 **`images/09-ssh-copy-id.png`** — `ssh-copy-id` réussi (`Number of key(s) added: 1`).

Vérification de l'authentification par clé (depuis SrvFichiers) :
```bash
ssh -i /root/.ssh/id_backup backupuser@192.168.56.20
```

📷 **`images/10-ssh-cle-test.png`** — Connexion réussie sans mot de passe.

> Clé dédiée à l'automatisation, sans phrase de passe : le script tourne la nuit, sans personne devant l'écran.

### 8.3 Script de sauvegarde (sur SrvFichiers)
```bash
nano /usr/local/bin/backup_samba.sh
```
```bash
#!/bin/bash
DATE=$(date '+%Y-%m-%d_%H-%M')
LOG=/var/log/backup_samba.log

echo "[$DATE] Debut de la sauvegarde" >> $LOG
rsync -avz --delete -e "ssh -i /root/.ssh/id_backup" \
  /srv/partages/ backupuser@192.168.56.20:/srv/sauvegardes/

if [ $? -eq 0 ]; then
  echo "[$DATE] Sauvegarde terminee avec succes" >> $LOG
else
  echo "[$DATE] ERREUR pendant la sauvegarde" >> $LOG
fi
```
```bash
chmod +x /usr/local/bin/backup_samba.sh
/usr/local/bin/backup_samba.sh   # test manuel
cat /var/log/backup_samba.log
```

📷 **`images/11-backup-script-test.png`** — Résultat du script + log affichant « Sauvegarde terminee avec succes ».

### 8.4 Planification (cron)
```bash
crontab -e
```
```
0 2 * * * /usr/local/bin/backup_samba.sh
```

📷 **`images/12-crontab-l.png`** — `crontab -l` confirmant la tâche planifiée.

---

## 9. Pare-feu UFW (SrvFichiers)

Exigence : le serveur ne doit pas être exposé directement à Internet. Même si Samba n'écoute que sur le réseau Host-only, un pare-feu explicite ajoute une seconde barrière indépendante de toute erreur de configuration future.

```bash
apt install ufw -y
ufw allow ssh                                              # TOUJOURS avant d'activer UFW
ufw allow from 192.168.56.0/24 to any port 445 proto tcp
ufw allow from 192.168.56.0/24 to any port 139 proto tcp
ufw enable
ufw status verbose
```

📷 **`images/13-ufw-status.png`** — `ufw status verbose` : pare-feu actif, SSH ouvert, Samba (445/139) restreint à `192.168.56.0/24`, `deny` par défaut en entrée.

> [!TIP]
> L'ouverture globale du port SSH (22) permet l'administration à distance depuis n'importe quel poste technique, tandis que Samba reste limité au réseau interne — application du principe du moindre privilège.

---

## 10. Tests finaux

### Test 1 : Accès Samba (autorisé / refusé)
Déjà couvert en partie 6 (captures 06 et 07).

### Test 2 : Quota dépassé
Sur le PC Windows :
```cmd
fsutil file createnew test.bin 600000000
```
Copier ce fichier de 600 Mo dans `\\192.168.56.10\Compta` avec le compte `ucompta` (limite hard = 500 Mo).

📷 **`images/14-quota-test-windows.png`** — Copie bloquée avec le message « espace insuffisant ».

### Test 3 : Log après exécution automatique (cron, pas manuelle)
```bash
date                    # heure actuelle
crontab -e              # remplacer temporairement l'heure par (heure actuelle + 2 min)
# attendre 2-3 minutes
cat /var/log/backup_samba.log
```

📷 **`images/15-cron-auto-log.png`** — Nouvelle ligne dans le log, horodatée automatiquement par cron.

> Remettre ensuite la ligne originale (`0 2 * * *`).

### Test 4 : Coupure réseau pendant la sauvegarde
Sur SrvSauvegarde :
```bash
nmcli connection down HostOnly-enp0s8
```
Sur SrvFichiers :
```bash
/usr/local/bin/backup_samba.sh
cat /var/log/backup_samba.log
```

📷 **`images/16-network-cut-log.png`** — Log affichant « ERREUR pendant la sauvegarde » (pas de plantage silencieux).

Puis réactivation et confirmation de la reprise :
```bash
# sur SrvSauvegarde
nmcli connection up HostOnly-enp0s8

# sur SrvFichiers
/usr/local/bin/backup_samba.sh
cat /var/log/backup_samba.log
```

📷 **`images/17-network-restored-log.png`** — Retour à « Sauvegarde terminee avec succes ».

---

## 11. Pistes d'amélioration

- Chiffrement du stockage sur SrvSauvegarde (le transit est déjà chiffré via SSH)
- Rotation des sauvegardes (conserver les 7 dernières nuits plutôt qu'un miroir simple)
- Alerte mail/webhook en cas d'échec de sauvegarde
- Haute disponibilité (bascule automatique si SrvFichiers tombe)
