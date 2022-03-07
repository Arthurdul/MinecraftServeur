# Compte rendu projet infra :

---

## I- Installation

``` 
arthur@arthur-VM:~$ sudo apt install openjdk-17-jre-headless curl screen nano bash grep wget
[...]
Complete!
arthur@arthur-VM:~$ sudo mkdir minecraft_server
arthur@arthur-VM:~$ sudo mkdir vanilla
```
Créer le sous-dossier vanilla permet de séparer les version (ex : version avec différents mode).

```
arthur@arthur-VM:~/minecraft_server/vanilla$ wget https://launcher.mojang.com/v1/objects/35139deedbd5182953cf1caa23835da59ca3d7cd/server.jar"
```
On récupère la conf fournie par Mojang
```
arthur@arthur-VM:~/minecraft_server/vanilla$ ls 
server.jar
arthur@arthur-VM:~/minecraft_server/vanilla$ sudo mv server.jar minecraft_server.1.18.1.jar
arthur@arthur-VM:~/minecraft_server/vanilla$ sudo nano minecraft
arthur@arthur-VM:~/minecraft_server/vanilla$ cat minecraft
java -Xmx2024 -Xms2024 -jar
arthur@arthur-VM:~/minecraft_server/vanilla$ sudo chmod a+x /home/arthur/minecraft_server/vanilla/minecraft
arthur@arthur-VM:~/minecraft_server/minecraft/vanilla$ ls -la | grep minecraft
-rwxr-xr-x 1 root root      112 févr. 15 01:31 minecraft
arthur@arthur-VM:~/minecraft_server/vanilla$ minecraft_server.1.18.1.jar nogui
arthur@arthur-VM:~/minecraft_server/vanilla$ sudo ./minecraft
[18:31:42] [main/ERROR]: Failed to load properties from file: server.properties
[18:31:43] [main/WARN]: Failed to load eula.txt
[18.31.44] [main/INFO]: You need to agree to the EULA in order to run the server. Go to eula.txt for more info.
```
Il faut maintenant éditer le fichier eula.txt apparu dans /home/arthur/minecraft_server/vanilla/
```
arthur@arthur-VM:~/minecraft_server/vanilla$ cat eula.txt
eula=true
```
On peut maintenant relancer le serveur :
```
arthur@arthur-VM:~/minecraft_server/vanilla$ sudo ./minecraft
[...]
[01:40:13] [Server thread/INFO]: Time elapsed: 319722 ms
[01:40:13] [Server thread/INFO]: Done (380.750s)! For help, type "help"
```
Le serveur est maintenant up.


## II- Backups

### 1. Le script :

On va maintenant écrire un script qui va permettre d'effectuer une sauvegarde de notre serveur :

```
arthur@arthur-VM:~$ sudo mkdir backups 
arthur@arthur-VM:~/backups$ sudo mkdir backup_minecraft
arthur@arthur-VM:~/backups/backup_minecraft$ sudo mkdir files 
arthur@arthur-VM:~/backups/backup_minecraft$ sudo nano backup.sh
arthur@arthur-VM:~/backups/backup_minecraft$ cat backup.sh
#!bin/bash
# Partie Backup
variabledate=$(date +"%m-%d-%y---%H-%M-%S")
file_name=$"Backup-$variabledate"
if [[ $(find /home/arthur/backups/backup_minecraft/files/ -type d -name "Backup*" | wc -l) != 0 ]]
then
        echo "Previous Backup found"
        rm -rf  /home/arthur/backups/backup_minecraft/files/
        mkdir /home/arthur/backups/backup_minecraft/files/$file_name
        chown admin-backup /home/arthur/backups/backup_minecraft/files/$file_name
        chmod 700 /home/arthur/backups/backup_minecraft/files/$file_name
else
        echo "No previous backup found, creating directory..."
        mkdir /home/arthur/backups/backup_minecraft/files/$file_name
        chown admin-backup /home/arthur/backups/backup_minecraft/files/$file_name
        chmod 700 /home/arthur/backups/backup_minecraft/files/$file_name
fi
cp -R /home/arthur/minecraft_server/minecraft/vanilla /home/arthur/backups/backup_minecraft/files/$file_name
# Partie Vérification
if [[ -d /home/arthur/backups/backup_minecraft/files/$file_name/vanilla ]]
then
        echo "Backup done!"
else
        echo "Failure during script execution."
fi
```

### 2. Sécuriser les backups :

Il faut maintenant créer un nouvel utilisteur qui sera le seul à avoir les perms pour exécuter le script (plus sécure) :

```
arthur@arthur-VM:~/backups/backup_minecraft$ sudo useradd -m admin-backup
arthur@arthur-VM:~/backups/backup_minecraft$ sudo passwd admin-backup
[sudo] password for arthur:
New password:
Retype new password:
passwd: password updated successfully
```

On change les permitions d'accès du script :

```
arthur@arthur-VM:~/backups/backup_minecraft$ sudo chown admin-backup backup.sh files/
arthur@arthur-VM:~/backups/backup_minecraft$ sudo chmod 700 backup.sh files/
arthur@arthur-VM:~/backups/backup_minecraft$ ls -la | tail -2
-rwx------ 1 admin-backup root 1030 févr. 15 00:47 backup.sh
drwx------ 3 admin-backup root 4096 févr. 15 02:10 files
```
Donc si j'essaye de m'introduire dans le dossier "files/" le shell me renvoie :

```
arthur@arthur-VM:~/backups/backup_minecraft$ cd files/
bash: cd: files/: Permission denied
```

Il en sera de même avec le script "backup.sh" seulement au lieu d'avoir un retour du shell il sera simplement impossible de lire le contenu du fichier si on n'est pas en root ou un sudoer.

### 3. Automatiser les backups :

Pour automatiser notre script, on va utiliser l'application crontab :

```
arthur@arthur-VM:~/backups/backup_minecraft$ sudo su
root@arthur-VM:/home/arthur/backups/backup_minecraft# crontab -u admin-backup -e
root@arthur-VM:/home/arthur/backups/backup_minecraft# exit
arthur@arthur-VM:~/backups/backup_minecraft$ su admin-backup -s /bin/bash
Password:
admin-backup@arthur-VM:~$ crontab -l  | grep backup.sh
*/10 * * * * bash /home/arthur/backups/backup_minecraft/backup.sh
```
