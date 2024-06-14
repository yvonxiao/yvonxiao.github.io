---
layout: post
title: ubuntu24.04 apt更新源后出现architecture错误信息[Resolved]
categories: Development
description: apt architecture error messages on Ubuntu 24.04.
keywords: apt, ubuntu24.04
---

修复新版apt更新源报错

### 问题描述

升级到ubuntu24.04后，之前装的msedge，chrome的apt源在`update`的时候总会报如下错误：

```bash
N: Skipping acquire of configured file 'main/binary-i386/Packages' as repository 'https://dl.google.com/linux/chrome/deb stable InRelease' doesn't support architecture 'i386'
N: Skipping acquire of configured file 'main/binary-i386/Packages' as repository 'https://packages.microsoft.com/repos/edge stable InRelease' doesn't support architecture 'i386'
```

看样子是没有指定架构，之前的`sources`文件应该是这样的

```bash
deb [arch=amd64] https://dl.google.com/linux/chrome/deb/ stable main
```

现在好像把`PGP`也合进来了，类似这样

```bash
Types: deb
URIs: https://dl.google.com/linux/chrome/deb/
Suites: stable
Components: main
Signed-By:
 -----BEGIN PGP PUBLIC KEY BLOCK-----
 -----END PGP PUBLIC KEY BLOCK-----
```

看样子没有加入之前的arch属性

### 问题解决

`sources`直接手动加入`Architectures: amd64`即可

```bash
Types: deb
Architectures: amd64
URIs: https://dl.google.com/linux/chrome/deb/
Suites: stable
Components: main
Signed-By:
 -----BEGIN PGP PUBLIC KEY BLOCK-----
 -----END PGP PUBLIC KEY BLOCK-----
```

如果需要批量修改，可以参考这个[脚本](https://gist.github.com/ThinGuy/21e7b3bb4b63404ad87541cd0ffebf09)。
