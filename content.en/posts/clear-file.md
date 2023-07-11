---
title: "Several methods to quickly reset a file to zero bytes"
date: 2023-07-10
tags: 
  - Shell
categories: 
  - Tips
toc: true
---

## Overview

Sometimes, you may need to reset a file to zero bytes without deleting it, such as in the case of a log file that is actively being used.

## Methods

Use the `truncate` command with the `-s 0` option, which specifies the size as zero bytes.

```shell
truncate -s 0 <filename>
```

Utilisez un espace réservé et un opérateur de redirection pour vider un fichier en l'écrasant.

```shell
:> <filename>
```
