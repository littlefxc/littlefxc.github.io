---
title: window删除右键管理员权限
date: 2019-05-06 19:06:00
categories: Windows
tags:
- 权限
---

## 删除右键管理员权限.reg

```bat
Windows Registry Editor Version 5.00
[-HKEY_CLASSES_ROOT\*\shell\runas]
[-HKEY_CLASSES_ROOT\Directory\shell\runas]
```