<!--
 * @Author: Yvon Xiao
 * @Date: 2020-03-23 21:46:05
 * @LastEditTime: 2020-03-23 22:46:43
 * @Description: In User Settings Edit
 * @FilePath: /yvonxiao.github.io/_posts/2020/2020-03-23-chrome-font-face-bug-on-ubuntu.md
 -->
---
layout: post
title: chrome在ubuntu环境里频繁出现错误日志
categories: Development
description: chrome font face bug on ubuntu
keywords: chrome, ubuntu
---

最近在我的ubuntu机器上查看系统日志的时候发现chrome抛出大量错误日志的原因

### 问题描述

偶然看了下`/var/log/syslog`，发现有大量的重复错误:
```ShellSession
org.gnome.Shell.desktop[1633]: [3123:1:0323/205829.164751:ERROR:child_process_sandbox_support_impl_linux.cc(79)] FontService unique font name matching request did not receive a response.
```
折腾了一番，定位在chrome上，而且尤其是打开google搜索相关的页面时，一定会抛出错误。

### 问题解决
判断是和`font service`相关，遂去`chrome://flags`页面搜索`font`，发现只有一个相关配置`#font-src-local-matching`，估计和字体搜索有关，我本地的网络应该是没问题的，需要访问的网站都能访问，不清楚具体原因，系统这样不断抛出错误，日志文件都浪费了不少空间。将上述配置设置成`Disabled`后，再看日志就没有这个错误了。