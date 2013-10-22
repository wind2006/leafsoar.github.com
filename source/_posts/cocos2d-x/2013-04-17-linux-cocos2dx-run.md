---
layout: post
title: Linux 环境 Cocos2d-x开发
date: 2013-4-17 17:30
comments: true
categories: Cocos2d-x
---

## Cocos2d-x 跨平台特性分析

作为一款跨平台的 2D 游戏引擎 [Cocos2d-x](http://cocos2d-x.org/) ，方便发布到各种移动平台，支持也在不断完善。可以跨平台运行，更据优势的是可以跨平台开发！

作为运行平台来说，目前主要以 **iOS** 和 **Android** 平台为多，对其它平台也有支持，如 **BlackBerry **平台，但基本只要满足前两者，就能达到我们跨平台运行的目的，这是由用户量决定的。

作为开发平台来说，常用的三种 **IDE**(集成开发环境) 开发方式：

* **Windows** 系统下使用 **Visual Stuido** 开发
* **Mac** 系统下使用 **Xcode** 开发
* **Linux** 系统下使用 **Eclipse + CDT** 开发

用过 Xcode 的人都说 Xcode 好用（ps:我没用过 :P），这是一套完整的开发环境，基于 **llvm** 的编译器，优秀的架构提供非常完善的工具链，先且不说，还有快速的模拟器，使开发过程流畅， Windows 平台的标准 IDE VS 也是易于使用，有 cocos2d-x 在 VS 中的项目模板，使开发简化了许多，并且直生成 Win32 可执行程序，即时看到运行效果。而使用 Eclipse 在 Linux 上开发 cocos2d-x 的人相对较少。并没有多少体会这样开发有什么优势！

<!-- more -->

以 Mac 用户来说，使用 cocos2d-x 很大原因是其跨平台（Android）的特性，否则有更为成熟的 **cocos2d-iphone** 可以使用，最后还是需要维护一个 Android 的开发环境，以方便移植。从 Windows 角度考虑，大多都是为了开发 Android 平台游戏，VS 作为开发来说是挺方便，但要编译到 Android 平台，就相当麻烦了，而这对于 Linux 的开发来说，相对容易，不需要开两个 IDE , VS 和 Eclipse 同时跑着了。

仁者见仁，智者见智，**用自己最熟悉的开发环境去写程序才能发挥应有的效率**。

-------------------------------------------------------------------------------

## 为什么使用 Linux 开发cocos2d-x

Linux 开发优势：

* 相比 Mac 下开发来说，开发成本低，普通 PC 机即可
* 相比 Windows 开发环境，只需要熟练使用一个 IDE Eclipse 即可
* Eclipse 作为默认的 Android 开发环境，总是不可避免要去使用
* gcc 编译器的编译异常信息比 VS 异常信息更容易找出问题 （个人感觉，VS 异常信息有如“天书”:P）
* 默认 UTF-8 编码，Windows 下开发 cocos2d-x 乱码解决起来麻烦，而 Linux 下，没有这个问题

Linux 开发劣势：

* **有所长必有所短～**

-------------------------------------------------------------------------------

## Linux 下怎样运行 cocos2d-x

要在 Linux 开发，我们首先要做的就是让 cocos2d-x 程序在 Linux 下跑起来。

开发机系统信息：

``` bash
	Debian 3.2.41-2 i686 GNU/Linux
	Debian/Wheezy testing
```	

cocos2d-x 当前稳定版本：`cocos2d-2.0-x-2.0.4`

下载地址：<http://cocos2d-x.googlecode.com/files/cocos2d-2.0-x-2.0.4.zip>

下载后解压，进入 **cocos2d-2.0-x-2.0.4** 目录执行脚本(编译过程需要检测依赖程序包，并且自动下载安装所需要的软件包，可以使用 **sudo** 提升权限运行)：

``` bash
	# [cocos2dx-path] 为 zip 解压后的目录 cocos2d-2.0-x-2.0.4 ，以后用此标示其项目目录
	cd [cocos2dx-path]
	
	./make-all-linux-project.sh    		# cocos2dx-path 当前目录执行命令
```

一会编译完毕，先不要问我这个脚本做了哪些事情，我们首先要做的就是把游戏跑起来，渐进式一点一点学习 cocos2d-x ~

``` bash
	cd [cocos2dx-path]/samples/HelloCpp/proj.linux/bin/release
	
	./HelloCpp			# 注意在当前目录执行 HelloCpp 以保证引用资源和库的相对路径正确

	# 如果出现类似一下错误，说明执行命令的路径不正确
	HelloCpp: error while loading shared libraries: libfmodex.so: cannot open shared object file: No such file or directory
```

**注意：** 在编译之前确保系统环境已经安装 gcc make 等程序， **Debian** 可以使用如下命令安装编译环境

``` bash
	sudo apt-get install build-essential
	
	gcc version 4.7.2 (Debian 4.7.2-5)
```	

-------------------------------------------------------------------------------

![Hello World](/images/2013/cocos2d-x-helloworld.jpg)

至此 cocos2d-x 自带的 **HelloCpp** 就已经能在 Linux 平台下运行了！

如果想看 cocos2d-x 具体能做哪些事情，可以看看 **TestCpp** 例子，里面包含了 cocos2d-x 的各种使用方法以及效果，这是一个非常实用的例子，如果有什么功能需要实现，就可以参考这个项目。

``` bash
	cd [cocos2dx-path]/samples/TestCpp/proj.linux/bin/release
	
	./TestCpp
```	

**工欲善其事 必先利其器**

后面将使用 Eclipse 来管理开发 cocos2d-x 项目 ~


