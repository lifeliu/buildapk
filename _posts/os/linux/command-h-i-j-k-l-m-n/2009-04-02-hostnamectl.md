---
layout: post
title: centos7永久更改主机名
categories: linux 
tags: hostnamectl
---

在centos7下更改主机名很简单，执行下面这个命令即可：

oldname ： 原主机名

newname ： 新主机名

```
[root@oldname ~]# hostnamectl set-hostname  newname
```