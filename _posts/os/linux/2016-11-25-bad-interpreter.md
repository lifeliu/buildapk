---
layout: post
title: ^M:bad interpreter:解决方法
subtitle: Each post also has a subtitle
author: fire
source: http://fireliu.com
categories: linux 
tags: shell
---

**^M: bad interpreter:解决方法**

在Linux中执行.sh脚本，异常提示/bin/sh^M: bad interpreter: No such file or directory。

分析：
这是不同系统编码格式引起的：在windows系统中编辑的.sh文件可能有不可见字符，所以在Linux系统下执行会报以上异常信息。

解决：

首先要确保文件有可执行权限

```
chmod a+x filename
```

然后修改文件格式 

```
vi filename
```

利用如下命令查看文件格式

```
:set ff 或 :set fileformat
```

可以看到如下信息 

```
fileformat=dos 或 fileformat=unix
```

利用如下命令修改文件格式 

```
:set ff=unix 或 :set fileformat=unix

:wq (存盘退出)
```

