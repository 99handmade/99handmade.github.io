---
title: "Backup MongoDB"
date: 2023-07-01
tags: 
  - Ubuntu
  - Cron
  - MongoDB
  - Docker
categories: 
  - Operations
toc: true
---

## Introduction

- Ubuntu
- MongoDB is deployed within a Docker container
- Cron task is used to run backup

When the database is running in a container, basically we will not install again MongoDB in the system. We can choose to either use a temporary database container or an existing one for performing the backup. The decision between these options depends on personal preference. In this article, I have opted to use a temporary database container.

Ubuntu Version

```txt
lsb_release -a

LSB Version: core-11.1.0ubuntu4-noarch:security-11.1.0ubuntu4-noarch
Distributor ID: Ubuntu
Description: Ubuntu 22.04.2 LTS
Release: 22.04
Codename: jammy
```

Bash Version

```txt
bash --version

GNU bash, version 5.1.16(1)-release (x86_64-pc-linux-gnu)
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
```

Docker Version

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

MongoDB Version

```txt
MongoDB shell version v4.4.19
MongoDB server version: 4.4.19
```

## Steps

### Create Backup Script

In this example script, MongoDB has more than one database, and each database has its own user and password protection. Backups are organized by week number and date within folders, and the backups are performed within the Docker container, directly writing to the local storage through a volume. It should be noted that in order to perform backups in another database container, the new container needs to connect to the existing Docker network.

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
COMPRESSED_FILE="$BACKUP_DIR/${BACKUP_NAME}.tar.gz"

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

# compress
echo "compressing backup directory..."
tar -czvf $COMPRESSED_FILE -C $BACKUP_DIR $BACKUP_NAME
echo "backup directory compressed to: $COMPRESSED_FILE"
echo ""

# remove uncompressed backup directory
echo "removing uncompressed backup directory..."
rm -rf $BACKUP_DIR/$BACKUP_NAME
echo "uncompressed backup directory removed."
echo ""

echo "$(date +"%Y-%m-%d %H:%M:%S") - Script execution complete."
```

### Create Cleanup Script

Keep only one week of backups and delete the rest.

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

### Modify Script Permissions

Cron executes the script under the user who created the task. The permissions can vary depending on the specific scenario. Here, we give the script execute permissions for all users.

```shell
chmod +x script.sh
```

### Create Cron File

There are several locations where Cron configurations can be placed. In this case, we create a file named "db" and place it in the `/etc/cron.d/` directory. The standard output and error streams of the script execution are redirected to log files. The backup script runs every day at 3 AM, and the cleanup script runs every Sunday at 4 AM, removing backups older than one week.

```shell
00 03 * * * root /bin/bash /opt/devops/mongodb/crons/backup_mongodb.sh >> /opt/devops/mongodb/crons/db_backup.log 2>&1
00 04 * * 7 root /bin/bash /opt/devops/mongodb/crons/cleanup_mongodb.sh >> /opt/devops/mongodb/crons/db_cleanup.log 2>&1
```
