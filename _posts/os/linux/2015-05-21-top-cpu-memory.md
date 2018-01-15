---
layout: post
title: Linux 查看CPU和内存使用前10的进程
categories: linux 
tags: cpu memory
---

**CPU前10的进程**

```

ps aux | head -1; ps aux | grep -v PID | sort -rn -k +3 | head

```

**内存前10的进程**

```

ps aux | head -1; ps aux | grep -v PID | sort -rn -k +4 | head

```