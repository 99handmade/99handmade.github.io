---
title: "备份 MongoDB"
date: 2023-07-01
tags: 
  - Ubuntu
  - Cron
  - MongoDB
  - Docker
categories: 
  - 运维
toc: true
---

## 简介

- Ubuntu操作系统
- MongoDB部署在Docker容器中
- 备份任务通过Cron执行

由于数据库部署在容器中, 一般在操作系统中, 就不会二次安装MongoDB, 我们可以选择单独运行临时的数据库容器, 也可以直接使用现有的数据库容器, 具体的选择是仁者见仁的问题. 个人而言, 我选择了前者.

Ubuntu版本

```txt
lsb_release -a

LSB Version: core-11.1.0ubuntu4-noarch:security-11.1.0ubuntu4-noarch
Distributor ID: Ubuntu
Description: Ubuntu 22.04.2 LTS
Release: 22.04
Codename: jammy
```

Bash版本

```txt
bash --version

GNU bash, version 5.1.16(1)-release (x86_64-pc-linux-gnu)
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
```

Docker版本

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

MongoDB版本

```txt
MongoDB shell version v4.4.19
MongoDB server version: 4.4.19
```

## 步骤

### 创建备份脚本

在这个实例脚本中, MongoDB有一个以上的数据库, 每个数据库都有各自的用户和密码保护. 备份按星期号, 日期组织在文件夹中, 备份在Docker容器中进行, 通过volume直接写在本机硬盘. 需要指出的是, 为了实现在另一个数据库容器中备份, 新容器需要连接到现有的Docker网络中.

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

### 创建清理脚本

只保留一周的备份, 其他的都删除.

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

### 修改脚本权限

定时任务以创建定时任务的用户身份执行脚本, 这个权限可以根据具体情况而定, 这里我们给脚本所有用户的执行权限.

```shell
chmod +x script.sh
```

### 创建Cron文件

有很多可以放Cron配置的地方, 这里我们创建一个文件名为db的文件  放到`/etc/cron.d/`文件夹里, 脚本执行的标准输出流和错误流, 重定向到日志文件. 每天凌晨三点执行一次备份, 每周日凌晨四点执行一次清理, 清除一周以前的备份.

```shell
00 03 * * * root /bin/bash /opt/devops/mongodb/crons/backup_mongodb.sh >> /opt/devops/mongodb/crons/db_backup.log 2>&1
00 04 * * 7 root /bin/bash /opt/devops/mongodb/crons/cleanup_mongodb.sh >> /opt/devops/mongodb/crons/db_cleanup.log 2>&1
```
