---
title: 如何删除正在运行的nginx的日志
date: 2020-03-05 11:23:01
tags:
- nginx
---

在删除nginx的日志的时候不要直接 `rm -f access.log`, 而是要 `echo "" > access.log`
