---
title: "Hugo"
date: 2021-09-16T16:48:29+08:00
draft: false
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Hugo 介绍

Hello, World!

# Hugo 主题

## book 主题

### **Install as git submodule**

Navigate to your hugo project root and run:

```
git submodule add https://github.com/alex-shpak/hugo-book themes/book
```

Then run hugo (or set theme = "book"/theme: book in configuration file)

hugo server --minify --theme book

### **Install as hugo module**

You can also add this theme as a Hugo module instead of a git submodule.

Start with initializing hugo modules, if not done yet:

hugo mod init github.com/repo/path
Navigate to your hugo project root and add [module] section to your config.toml:

```toml
[module]
[[module.imports]]
path = 'github.com/alex-shpak/hugo-book'
```

Then, to load/update the theme module and run hugo:

```sh
hugo mod get -u
hugo server --minify
```