---
layout: post
title: package-cleanup命令详解
categories: linux 
tags: package-cleanup yum
---

**package-cleanup**

用于清理本地安装的RPM软件包

```xml

    package-cleanup --leaves

```

列出与其他RPM没有依赖关系的软件包，又叫叶节点（leaf node），即，没有软件包依赖叶节点。

```xml

    package-cleanup --orphans

```

列出当前软件仓库中不再提供支持的本地已安装的软件包。也就是说，列出的软件包将不会再升级。

```xml

    package-cleanup --oldkernels

```

删除旧内核文件（kernel, kernel-devel）。

```xml

    package-cleanup --oldkernels --ount=3 --keepdevel

```

含义是：保留最近3个内核文件和kernel-devel文件，并删除其余的kernels。

```xml

    package-cleanup --problems

```

列出有依赖问题的软件包。

```xml

    package-cleanup --dupes

```

扫描重复安装的RPM软件包。

```xml

    package-cleanup --cleandupes

```

扫描重复安装的软件包，并删除老版本的软件包。

```xml

    yum-complete-transaction

```

使用指令4、6
