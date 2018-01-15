---
layout: post
title: linux下cat,tac,rev常见用法
categories: linux 
tags: cat tac rev
---

**cat**

格式：cat [op] file

功能：

1. 一次显示整个文件:cat filename
2. 从键盘创建一个文件:cat > filename 只能创建新文件,不能编辑已有文件.
3. 将几个文件合并为一个文件:cat file1 file2 > file


说明：

　　op为命令参数，常用的有-b、-n、-s

* -b：对非空输出行编号
* -n：对输出的所有行编号,由1开始对所有输出的行数编号
* -s：有连续两行以上的空白行，就代换为一行的空白行