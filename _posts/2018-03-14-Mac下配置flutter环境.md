---
layout: post
title: "Mac下配置flutter环境 "
author: "Gerald"
---

## Get Started
安装环境: Mac OS 10.12.6

### 下载flutter

~~~shell
$ git clone -b alpha https://github.com/flutter/flutter.git
~~~
<!--more-->

设置环境变量
~~~shell
$ export PATH=$HOME/flutter/bin:$PATH
$ source ~/.bash_profile
~~~

重新打开`terminal`,输入 `flutter`
OK

### 检查依赖

~~~shell
$ flutter doctor
~~~
如果你在大陆不出意外的话会在：`Downloading Gradle Wrapper…`的时候卡住，看log被墙了是。虽然我一直挂着SS，无奈`terminal`不走代理
[问题详情在此](https://github.com/flutter/flutter/issues/11674)

解决方案
1. [为teminal开启proxy](http://www.jianshu.com/p/32dfb5289cf5)
2. [浏览器手动下载](http://blog.csdn.net/xjwangliang/article/details/78042740) (亲测可用)
3. 另寻梯子

再次运行`flutter doctor` 按照提示一步步安装所需依赖可能需要安装：
- Xcode
- Android (Android sdk)
- ios-deploy

不再一一赘述

### IDE

`flutter`不限编辑器，官方推荐使用`IntelliJ`系列，因为习惯我选择用`Atom`，看了下`Atom`下也有支持`flutter`开发的插件 `flutter package`， 安装如下:
1. `shift+commnd+p` 输入 `install` 选择`install packages and themes`
2. 搜索 `flutter` 并安装
3. `Packages > Flutter > Packages Settings`，设置`FLUTTER_ROOT`为Flutter SDK的根目录的路径
4. `Packages > Dart > Packages Settings`，设置`Dart SDK Location`为Flutter SDK的根文件夹下的`bin/cache/dart-sdk`(默认自动设置)

over~
