---
title: "Backup MongoDB"
date: 2023-07-01
tags:
  - Ubuntu
  - Cron
  - MongoDB
  - Docker
categories:
  - Gestion des opérations
toc: true
---

## Problématique

- System d'exploitation est Ubuntu
- MongoDB est déployé dans un conteneur Docker
- Une Cron tâche est planifiée pour exécuter le backup script

Lorsque la base de données est exécutée dans un conteneur, normalment nous n'avons pas reinstaller le double MongoDB dans OS, à cette condition, nous avons le choix entre utiliser un conteneur de base de données temporaire ou le conteneur existant pour effectuer la sauvegarde. Le choix entre ces options dépend des préférences personnelles. Dans cet article, nous prenons un conteneur de base de données temporaire.

Version Ubuntu

```txt
lsb_release -a

LSB Version: core-11.1.0ubuntu4-noarch:security-11.1.0ubuntu4-noarch
Distributor ID: Ubuntu
Description: Ubuntu 22.04.2 LTS
Release: 22.04
Codename: jammy
```

Version Bash

```txt
bash --version

GNU bash, version 5.1.16(1)-release (x86_64-pc-linux-gnu)
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
```

Version Docker

```txt
Client: Docker Engine - Community
 Version:           24.0.1
 API version:       1.43
 Go version:        go1.20.4
 Git commit:        6802122
 Built:             Fri May 19 18:06:21 2023
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          24.0.1
  API version:      1.43 (minimum version 1.12)
  Go version:       go1.20.4
  Git commit:       463850e
  Built:            Fri May 19 18:06:21 2023
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.21
  GitCommit:        3dce8eb055cbb6872793272b4f20ed16117344f8
 runc:
  Version:          1.1.7
  GitCommit:        v1.1.7-0-g860f061
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

Version MongoDB

```txt
MongoDB shell version v4.4.19
MongoDB server version: 4.4.19
```

## Étapes

### Création du script de sauvegarde

Dans ce script exemple, MongoDB dispose de plusieurs bases de données, chacune étant protégée par ses propres identifiants d'utilisateur et mot de passe. Les sauvegardes sont organisées par numéro de semaine et date, stockées dans un dossier à l'intérieur du conteneur Docker, puis écrites sur le disque local via un volume. Il est important de noter que pour effectuer la sauvegarde dans un autre conteneur de base de données, le nouveau conteneur doit être connecté au réseau Docker existant.

```bash
#!/bin/bash

# display environment information
echo "$(date +"%Y-%m-%d %H:%M:%S") - Starting the script..."
echo "Current Date and Time: $(date)"
echo "Current User: $(whoami)"
echo "Current Directory: $(pwd)"
echo "Login Shell: $SHELL"
echo ""

# display system information
echo "Operating System: $(uname -s)"
echo "Hostname: $(hostname)"
echo "Kernel Version: $(uname -r)"
echo "Processor Architecture: $(uname -m)"
echo ""

# def
DATE=$(date +%d%m%Y_%H%M)
WEEK=$(date +%W)
BACKUP_DIR="/data/db_backup/$WEEK"
BACKUP_NAME="$DATE"

# directory permission
mkdir -p $BACKUP_DIR/$BACKUP_NAME
chown -R 999:999 $BACKUP_DIR/$BACKUP_NAME

# define database credentials
declare -A credentials=(
  ["db_name_1"]="username_1:password_1"
  ["db_name_2"]="username_2:password_2"
)

# run backup

for db_name in "${!credentials[@]}"; do
  echo ">>>>>>>>> start to backup $db_name >>>>>>>>>"
  IFS=':' read -r username password <<< "${credentials[$db_name]}"
  docker run -u $(id -u):$(id -g) --rm --network mongodb_mongo_net -v $BACKUP_DIR/$BACKUP_NAME:/workdir -w /workdir \
    mongo:4.4.19 mongodump --out /workdir \
    --db=$db_name --username $username --password $password --host=bewellconnect-mongodb --authenticationDatabase=$db_name
  echo "<<<<<<<<< end to backup $db_name <<<<<<<<<"
  echo ""
done

echo "$(date +"%Y-%m-%d %H:%M:%S") - Script execution complete."
```

### Création du script de nettoyage

Ce script permet de ne conserver que les sauvegardes d'une semaine, en supprimant les autres.

```bash
#!/bin/bash

# display environment information
echo "$(date +"%Y-%m-%d %H:%M:%S") - Starting the script..."
echo "Current Date and Time: $(date)"
echo "Current User: $(whoami)"
echo "Current Directory: $(pwd)"
echo "Login Shell: $SHELL"
echo ""

# display system information
echo "Operating System: $(uname -s)"
echo "Hostname: $(hostname)"
echo "Kernel Version: $(uname -r)"
echo "Processor Architecture: $(uname -m)"
echo ""

#def
BACKUP_DIR="/data/db_backup"

# get current week number
current_week=$(date +%W)

# run cleanup
for dir in "$BACKUP_DIR"/*/; do
  dir_name=$(basename "$dir")
  if [ "${dir_name}" \< "${current_week}" ]; then
    echo "delete backup: $dir"
    rm -rf "$dir"
  fi
done

echo "$(date +"%Y-%m-%d %H:%M:%S") - Script execution complete."
```

### Modification des autorisations de script

Les tâches planifiées sont exécutées en tant qu'utilisateur qui a créé la tâche planifiée. Les autorisations peuvent varier en fonction de la situation spécifique. Ici, nous donnons à tous les utilisateurs l'autorisation d'exécuter le script.

```shell
chmod +x script.sh
```

### Création du fichier Cron

Il existe de nombreux endroits où vous pouvez placer la configuration Cron. Ici, nous allons créer un fichier appelé "db" et le placer dans le dossier `/etc/cron.d/`. Les sorties standard et les erreurs du script seront redirigées vers un fichier journal. Une sauvegarde sera effectuée tous les jours à 3 heures du matin, et une opération de nettoyage sera effectuée tous les dimanches à 4 heures du matin, supprimant les sauvegardes datant d'une semaine.

```shell
00 03 * * * root /bin/bash /opt/devops/mongodb/crons/backup_mongodb.sh >> /opt/devops/mongodb/crons/db_backup.log 2>&1
00 04 * * 7 root /bin/bash /opt/devops/mongodb/crons/cleanup_mongodb.sh >> /opt/devops/mongodb/crons/db_cleanup.log 2>&1
```
