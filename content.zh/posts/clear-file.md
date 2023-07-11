---
title: "快速重制文件为0字节的几种方法"
date: 2023-07-10
tags: 
  - Shell
categories: 
  - Tips
toc: true
---

## 简介

有些时候, 你需要把一个文件重制为0字节, 但是又不能删除它, 比如一个正在使用的日志文件.

## 几种方法

使用截断命令`truncate`, 参数`-s 0`指定大小为0字节

```shell
truncate -s 0 <filename>
```

使用占位符和重定向操作符, 就是把一个空操作符覆盖写入文件

```shell
:> <filename>
```
