# Streaming Audio

Notre projet à pour but de pouvoir écouter de la musique sur un site autre que Deezer, Spotify, Soundcloud. Pour cela nous utiliserons Plex sous [Debian](https://www.linuxvmimages.com/images/debian-10/)

## Membres du projet

- Bastien AUMENIER

- Pierre-Louis BERTIN

- Hugo JOYET

- Manoa MARCHAND

## Sommaire

- Backup

- RAID

- Monitoring

- Plex

## Installation

### Backup

Voici les étapes à suivre pour installer une backup sur la même machine :

- Premièrement actualiser les packages

```bash
sudo apt update
```
- Nous allons utiliser le package 
[borgbackup](https://borgbackup.readthedocs.io/en/stable/)

```bash
sudo apt install borgbackup
```
- Maintenant, il faut initialiser le dossier où toutes les backups seront sauvegardées

```bash
borg init -e repokey /path-to-repo
```
Ici on préfèrera utiliser 
[repokey](https://borgbackup.readthedocs.io/en/stable/usage/init.html)
pour l'encryptage du dossier

- Passons à la création d'un script afin de générer une backup périodiquement

```bash
nano /path-to-script
```
remplissez le avec :

```bash
borg create -v "/path-to-repo::name-for-backup" /folder-to-backup
```
- Permettre l'exécution du script

```bash
chmod +x /path-to-script
```
- Et pour finir on utilisera 
[cron](https://doc.ubuntu-fr.org/cron) 
pour automatiser les backups tant que la machine est allumée, pour cela nous allons édité notre `crontab` :

```bash
crontab -e 
```
et y ajouter une nouvelle ligne qui permettra d'effectuer une backup toute les heures

```bash
00 */1 * * * /path-to-script
```
Désormais si tout s'est bien passer, votre backup est mise en place

### Raid 1 / mdadm

L'exemple se base sur une machine virtuelle disposant d'un disque dur constitué de deux partitions (/dev/sda1 pour le système et /dev/sda5 pour le swap) auquel est adjoint un second disque dur de même capacité (/dev/sdb).

Nous allons d'abord préparer le second disque :
- Se connecter en super-utilisateur :

```bash
su -
```
- Installer mdadm, l'outil de gestion de volumes RAID logiciels :

```bash
apt update
```
```sh
apt install mdadm
```

- Cloner la table de partitions de /dev/sda sur /dev/sdb :

```bash
sfdisk -d /dev/sda | sfdisk --force /dev/sdb
```
- Changer l'étiquette des partitions de /dev/sdb en « RAID Linux » :

```bash
fdisk /dev/sdb
```
Taper les commandes suivantes :

```
Commande (m pour l'aide) : t # Changer les étiquettes
Numéro de partition (1,2,5, 5 par défaut) : 1
Code Hexa (taper L pour afficher tous les codes) : fd
Commande (m pour l'aide) : t
Numéro de partition (1,2,5, 5 par défaut) : 5
Code Hexa (taper L pour afficher tous les codes) : fd
Commande (m pour l'aide) : w # Appliquer
```
#### Créer le RAID

- Créer les volumes RAID 1 sans les partitions /dev/sda — lesquelles seront ajoutées ultérieurement :

```sh
mdadm --create /dev/md0 --level=1 --raid-disks=2 missing /dev/sdb1
```
```sh
mdadm --create /dev/md1 --level=1 --raid-disks=2 missing /dev/sdb5
```

- Formater /dev/md0 en ext4 et /dev/md1 en swap :

```
mkfs.ext4 /dev/md0
```
```sh
mkswap /dev/md1
```
- Déclarer les nouveaux volumes RAID 1 créés dans le fichier de configuration :

```sh
mdadm --examine --scan >> /etc/mdadm/mdadm.conf
```

#### Préparer le système existant à rejoindre le RAID

- Changer l'attribution des partitions / et swap pour /dev/md0 et /dev/md1 :

```sh
nano /etc/fstab
```
Modifier les lignes par :

```sh
/dev/md0    /       ext4    errors=remount-ro   0       1
/dev/md1    none    swap    sw                  0       0
```

- Configurer temporairement Grub afin de permettre le démarrage du système sur le RAID :

```sh
nano /etc/grub.d/1_raid1
```
Y coller ceci :

```sh
#!/bin/sh
exec tail -n +3 $0
menuentry 'RAID 1' --class debian --class gnu-linux --class gnu --class os {
    insmod gzio
    insmod raid
    insmod mdraid1x
    insmod part_msdos
    insmod ext2
    set root='(md/0)'
    linux /boot/vmlinuz-4.19.16-4-amd64  root=/dev/md0   ro  quiet
    initrd /boot/initrd.img-4.19.16-4-amd64
}
```
Rendre le fichier exécutable :

```sh
chmod +x /etc/grub.d/1_raid1
```

- Désactiver l'usage de l'UUID par Grub :

```sh
nano /etc/default/grub
```

Décommenter la ligne suivante :

```sh
#GRUB_DISABLE_LINUX_UUID=true
```

- Mettre à jour la configuration de Grub :

```sh
update-grub
```
```sh
update-initramfs -u
```

#### Copier le contenu du premier disque sur le RAID

- Créer un point de montage pour /dev/md0 :

```sh
mkdir /mnt/md0
```

- Monter le volume correspondant :

```sh
mount /dev/md0 /mnt/md0
```

- Copier l'intégralité de / sur /dev/md0 :

```sh
cp -apx / /mnt/md0
```

- Redémarrer le système, qui devrait se lancer sur le RAID :

```sh
reboot
```

#### Ajouter le premier disque au RAID

Une fois le système redémarré sur RAID 1, 
- Se connecter en super-utilisateur :

```sh
su -
```

- Changer l'étiquette des partitions de /dev/sda en « RAID Linux » :

```sh
fdisk /dev/sda
```
Taper les commandes suivantes :

```
Commande (m pour l'aide) : t 
Numéro de partition (1,2,5, 5 par défaut) : 1
Code Hexa (taper L pour afficher tous les codes) : fd
Commande (m pour l'aide) : t
Numéro de partition (1,2,5, 5 par défaut) : 5
Code Hexa (taper L pour afficher tous les codes) : fd
Commande (m pour l'aide) : w
```
- Ajouter les partitions de /dev/sda aux volumes RAID :

```sh
mdadm --add /dev/md0 /dev/sda1
mdadm --add /dev/md1 /dev/sda5
```
- Surveiller la progression de la synchronisation des volumes RAID :

```sh
watch cat /proc/mdstat
```
Une fois l'opération effectuée, quitter avec Ctrl+C.

#### Finaliser la configuration du nouvel environnement
 
- Supprimer la configuration manuelle de Grub :

```sh
rm /etc/grub.d/1_raid1
```
- Réactiver l'usage de l'UUID par Grub :

```sh
nano /etc/default/grub
```
Commenter la ligne suivante :

```sh
GRUB_DISABLE_LINUX_UUID=true
```
- Mettre à jour la configuration de Grub :

```sh
update-grub
```
```sh
update-initramfs -u
```
- Installer Grub sur les deux systèmes de fichiers, /dev/sda et /dev/sdb :

```sh
grub-install --recheck /dev/sda
```
```sh
grub-install --recheck /dev/sdb
```

- Supprimer le point de montage /mnt/md0 :

```sh
rmdir /mnt/md0
```
- Et pour finir redémarrer le système :

```sh
reboot
```

Si tout s'est bien passer vous devriez avoir votre RAID 1 de mise en place

PS: C'est juste avant l'étape d'ajouter le premier disque sur le RAID que nous avions eu des problèmes

### Monitoring

Pour installer Netdata, y compris toutes les dépendances requises pour se connecter à Netdata Cloud, et obtenir des updates automatiques, exécutez ce qui suit en tant qu'utilisateur normal :

```sh
bash <(curl -Ss https://my-netdata.io/kickstart.sh)
```
Pour obtenir plus d'informations sur ce script d'installation, notamment sur la manière de désactiver les mises à jour automatiques ou de désactiver les statistiques anonymes, consultez le fichier 
[kickstart.sh](https://learn.netdata.cloud/docs/agent/packaging/installer/methods/kickstart)

### Plex for Debian

Nous allons utiliser le dépôt officiel de Plex. 
- Commencez par importer la clé GPG du dépôt.

```sh
sudo apt install curl

curl https://downloads.plex.tv/plex-keys/PlexSign.key | sudo apt-key add -
```
- Ajoutez le dépôt Plex APT à la liste des dépôts de logiciels de votre système :

```sh
echo deb https://downloads.plex.tv/repo/deb ./public main | sudo tee /etc/apt/sources.list.d/plexmediaserver.list

```

- Une fois le dépôt Plex activé, mettez à jour la liste des paquets et installez la dernière version du serveur multimédia Plex avec :

```sh
sudo apt install apt-transport-https
sudo apt update
sudo apt install plexmediaserver
```

- Pour vérifier que l'installation a réussi et que le service SSH est en cours d'exécution, tapez la commande suivante :

```sh
sudo systemctl status plexmediaserver
```

#### Configuration de Plex Media Server

- Avant de lancer l'assistant d'installation de Plex, créons les répertoires qui stockeront les fichiers multimédia de Plex :

```sh
sudo mkdir -p /opt/plexmedia/folder-to-Musique
```
- Le serveur multimédia Plex s'exécute sous l'utilisateur "plex" qui doit avoir les droits de lecture et d'exécution sur les fichiers et répertoires. Pour définir la propriété correcte, exécutez la commande suivante.

```sh
sudo chown -R plex: /opt/plexmedia
```

Nous pouvons maintenant procéder à la configuration du serveur. Ouvrez votre navigateur, tapez `http://YOUR_SERVER_IP:32400/web` et suivez les instructions

### Votre Plex server est désormais fonctionnel
