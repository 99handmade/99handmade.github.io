---
title: "Comment remettre un fichier à zéro octet"
date: 2023-07-10
tags: 
  - Shell
categories: 
  - Tips
toc: true
---

## Aperçu

Parfois, il est nécessaire de réinitialiser un fichier à 0 octet sans le supprimer, par exemple, un fichier journal en cours d'utilisation.

## Méthodes

Utilisez la commande `truncate` avec le paramètre `-s 0` pour spécifier une taille de 0 bytes.

```shell
truncate -s 0 <nom du fichier>
```

使用占位符和重定向操作符, 就是把一个空操作符覆盖写入文件

```shell
:> <nom du fichier>
```
