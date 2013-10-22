---
layout: post
title: Linux 下开发 Android Cocos2d-x 游戏
date: 2013-4-22 22:15
comments: true
categories: Cocos2d-x
---

开发 Android 应用的首选 IDE 是什么，是 Eclipse ，而开发 Android 平台的 cocos2d-x 游戏呢，当然也是 Eclipse 。


##Eclipse Android 开发环境搭建

* 如果你的 Eclipse 不支持 Java 开发，请安装相关插件，或者使用 [Eclipse IDE for Java Developers](http://www.eclipse.org/downloads/) 版本集成的 Java 开发环境。
* 仅仅 Java 开发环境还不够，我们还需要 [ADT](http://developer.android.com/tools/sdk/eclipse-adt.html)(Android Development Tools) ，顾名思义，通过它，我们才能在 Eclipse 建立 Android 项目，与 Eclipse 集成，打包生成 Android 应用程序包 Apk 文件。
* 作为 ADT 的依赖环境，[Android SDK](http://developer.android.com/sdk/index.html) 也是必须的。
* 由于 cocos2d-x 是由 C/C++ 编写，为了编译 Android 平台 C/C++ 程序，我们还需要安装 [NDK](http://developer.android.com/tools/sdk/ndk/index.html#Downloads)， 注意选择  Linux 版本， 32 位或者 64 位视开发机而定。

<!-- more -->

安装好 Eclipse、ADT 和 Android SDK、NDK，打开 Eclipse 做些简单的配置，打开 Eclipse:

``` bash
	Eclipse -> Window -> Preferences -> Android
```	

* 在右侧设置 **SDK Location** 为你的 Android SDK 位置。只有当位置正确，才能正确点击 OK。
* Linux 下进入 Android SDK 目录，运行 `[android-sdk-linux]/tools/android` ，程序，安装相关工具，各种版本的 Android SDK Platform，代码实例等，可以选择 4.1.2 版本。
* Eclipse 切换到 Java Perspective (Eclipse 右上角设置)，点击 Eclipse 菜单 `Window -> Android Virtual Device Manager` 创建一个虚拟机，版本对应上一步骤所安装的 SDK 版本。
* 为了方便起见，我们将 Android SDK 和 NDK 的执行命令等（`[android-sdk-linux/tools] 和 ndk-build`），添加到 Linux 系统 PATH 环境变量。

此时，通过安装配置，要能**完成在 Linux 下使用 Eclipse 开发 Android 应用和 Android Jni 应用**，我们才能继续接下来的过程，如果配置出现问题，不能开发 Android 或者 Jni 应用，那么请先解决之 ~

环境搭建过程，如果出现什么问题或者疑问，可以 [Google](http://www.google.com) 之，或者在博客之后留言交流 ~

##创建 Linux Cocos2d-x for Android 项目

我们首先通过 cocos2d-x 自带的创建脚本创建 Android cocos2d-x 项目。

``` bash
	cd [cocos2d-x-path]
	./create-android-project.sh # 回车
	...
	Input package path. For example: org.cocos2dx.example
	com.leafsoar.HelloWorld    # 输入包名
	...
	...
	----------
	id: 7 or "android-16"
	     Name: Android 4.1.2
	     Type: Platform
	     API level: 16
	     Revision: 3
	     Skins: QVGA, WVGA800 (default), WSVGA, HVGA, WQVGA432, WVGA854, WQVGA400, WXGA720, WXGA800, WXGA800-7in
	     ABIs : armeabi-v7a
	input target id:
	7    # 输入 target id，参照上面，根据自己的 sdk 版本选择
	...
	input your project name:
	HelloWorld     		# 输入项目名称
```	

会在 `[cocos2d-x-path]` 目录创建一个 `HelloWorld` 的目录，里面包含文件如下：

``` bash
	HelloWorld/
	├── Classes
	│   ├── AppDelegate.cpp
	│   ├── AppDelegate.h
	│   ├── HelloWorldScene.cpp
	│   └── HelloWorldScene.h
	├── proj.android
	│   ├── AndroidManifest.xml
	│   ├── ant.properties
	│   ├── bin
	│   ├── build_native.sh
	│   ├── build.xml
	│   ├── jni
	│   ├── libs
	│   ├── local.properties
	│   ├── ndkgdb.sh
	│   ├── proguard-project.txt
	│   ├── project.properties
	│   ├── res
	│   └── src
	└── Resources
	    ├── CloseNormal.png
	    ├── CloseSelected.png
	    └── HelloWorld.png
```		

[上篇博文](/archives/2013/04-19-11.html)我们简单分析了一下这样的目录组织方式，和其共用。创建好 cocos2d-x android 项目，我们开始编译，`proj.android`目录下提供一个自动编辑脚本，完成的工作是，编译 cocos2d-x C/C++ 程序，并且在 libs 目录生成相应的 **so** 库文件。

``` bash
	cd [HelloWorld]/proj.android
	././build_native.sh
	...
	...
	...
	StaticLibrary  : libextension.a
	SharedLibrary  : libgame.so
	Install        : libgame.so => libs/armeabi/libgame.so
	make: Leaving directory '[cocos2dx-path]/HelloWorld/proj.android'
```	

看到如上信息，说明已经编译成功，首次编译过程稍微缓慢一点，如果没有出现以上结果，或者编译错误，多是因为系统环境没有配置正确。需要环境变量 `NDK_ROOT`、`COCOS2DX_ROOT`等，根据提示具体分析。

##配置集成开发环境

为了方便项目管理，我们把 `HelloWorld` 目录移动到 Ecilpse 的工作空间之中，默认实在 [cocos2dx-path] 目录。然后使用 Eclipse 导入本地项目，指定 [HelloWorld/proj.android] 目录导入项目。

* 导入之后我们发现项目出现感叹号，这是因为 Eclipse 管理此项目的依赖项目并不存在，我们继续用 Eclipse 导入 `[cocos2dx-path]/cocos2dx/platform/android/java` **libcocos2dx** 项目。
* 我们右击 HelloWorld 修改项目属性，找到 Android  项，看到 Library 属性框，所引用的项目出现问题，首先 remove 然后 重新 Add 选择刚才添加的 libcocos2dx 项目。
* 我们 clean 清空一下 HelloWorld  项目。
* 此时在我的项目中 **AndroidManifest.xml** 文件出现 `androi:icon` 图标的错误，修改成 `@drawable/ic_launcher`
* 右击项目 **Run as -> Android Application** 就能运行此应用了

**以上问题视自己的具体情况而定，可能出现不同的错误。**

此时我们发现 Android 项目中，**bin** 目录生成了 HelloWorld.apk 文件，这个就是最终的 Android 平台 cocos2d-x 游戏安装包了。并且可以运行。

至此，我们已经能够在 Linux 上运行编译运行 Android 平台的 cocos2d-x 游戏，但是每次我们在编写完 cocos2d-x 游戏源代码，我们都要执行 `build_native.sh` 命令，当然可以简化此步骤，并且将上一偏博文的的 Linux 下开发 Cocos2d-x 游戏，一遍编写调试，一边运行在 Android 平台，或者两个项目同时进行。这些内容将会在下一篇文章之中 ~

至此还没有完，因为我们移动了项目目录，所以再次执行 `build_native.sh` 命令会出现问题，我们需要修改此文件中的配置：

``` bash
	COCOS2DX_ROOT="$DIR/../.."
```	

将此值设置为 [cocos2dx-path] 的绝对目录，或者使用 COCOS2DX_ROOT 的环境变量。以保证此命令能够执行编译成功。这个问题留给读者自己解决吧 : p
