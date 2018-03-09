---
layout: post
title: CentOS远程SSH登录
subtitle: CentOS 6.X ssh root login
categories: linux 
tags: network ssh
---

#1. 禁止ROOT远程SSH登录
vi /etc/ssh/sshd_config

修改
PermitRootLogin yes
为
PermitRootLogin no

重启sshd服务
service sshd restart

远程管理用普通用户登录，然后用su root切换到root

#2. 修改ssh默认端口
vi /etc/ssh/sshd_config

修改
port 51822

重启sshd服务
service sshd restart

#3. 指定端口ssh登录
```
ssh user@192.168.1.1 -p 51822
```