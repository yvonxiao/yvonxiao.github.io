---
layout: post
title: electron打包的应用在ubuntu24.04上无法启动[Resolved]
categories: Development
description: Resolved the issue of Electron-packaged applications failing to start on Ubuntu 24.04.
keywords: electron, ubuntu24.04
---

暂时解决electron打包的应用在ubuntu24.04上无法启动的问题

### 问题描述

今天偶然敲了下`do-release-upgrade -d`，发现可以直接升级到`ubuntu24.04 LTS`了，就兴冲冲地升到最新版。结果发现之前下的`Obsidian`，`Chatbox`这类electron打包的都不能打开了，错误日志类似输出：

```shell
The SUID sandbox helper binary was found, but is not configured correctly. Rather than run without sandboxing I'm aborting now. You need to make sure that /tmp/.mount_ObsidiEJ39h1/chrome-sandbox is owned by root and has mode 4755.
```

当然每个应用的sandbox位置不一样。

如果就是按照这个输出结果去查询，会得到下面两个解决方案：

- 加入启动参数`--no-sandbox`
- 修改内核参数`sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0`

我都觉得不好，第一个方案关闭了沙盒，这本来是Chromium内核自带的安全方案，为啥要取消？第二个修改内核会影响系统所有的服务，改动太大了，我只是跑几个独立app而已。

从第二个方案发现这个问题的出现其实就是ubuntu24.04的新特性`apparmor`的默认安全策略带来的，所以在官方打包程序没有自动添加`apparmor`的profile的情况下，就要手动去添加profile指定用户命名空间等权限。另外，我发现obsidian的AppImage里做好的desktop文件里，启动参数是添加了`--no-sandbox`，动手能力强的倒是可以把AppImage修改成带指定参数(比如代理)和图标的完整版。不过这里我就简单让类似应用能够跑起来就行了。

### 问题解决

如果安装了最新版的`vscode`的话，会发现`/etc/apparmor.d/`下对应的`code`的profile文件，就照着给个简单的命名空间权限就好了。

vs code

```bash
# This profile allows everything and only exists to give the
# application a name instead of having the label "unconfined"

abi <abi/4.0>,
include <tunables/global>

profile vscode /usr/share/code{/bin,}/code flags=(unconfined) {
  userns,

  # Site-specific additions and overrides. See local/README for details.
  include if exists <local/code>
}
```

就照着这个格式，修改下名和路径即可。比如我就手动的添加了obsidian和chatbox的profile文件。

执行`systemctl restart apparmor`，再次打开各自的应用，都没有问题了。

`apparmor`的profile配置可以参考官方文档，如果需要很多权限，都需要一一配置，比如这里贴一个`docker-desktop`的

```bash
# This profile allows everything and only exists to give the
# application a name instead of having the label "unconfined"

abi <abi/4.0>,
include <tunables/global>

#后面的地址按照自己的 docker-desktop的实际目录填写
profile docker-desktop  /opt/docker-desktop/bin/*  flags=(unconfined) {
  userns,
  capability,
  capability chown,
  capability dac_override,
  capability setuid,
  capability setgid,
  capability net_bind_service,

  # Site-specific additions and overrides. See local/README for details.
  include if exists <local/docker-desktop>
}
```

这次breaking change带来的问题应该是暂时的，后面各家都会出修复方案。

### 参考

- [1] [docker desktop ubuntu24.04 无法启动](https://juejin.cn/post/7376556275972522047)