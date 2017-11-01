---
layout: post
title: CentOS 7 防火墙添加查看端口
author: fire
categories: linux 
tags: firewall
---

**查看端口**

```
firewall-cmd --list-port
```

**添加端口**

```
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --zone=public --add-port=8080-8090/tcp --permanent
```

**重载规则**

```
firewall-cmd --reload
firewall-cmd --complete-reload
```
