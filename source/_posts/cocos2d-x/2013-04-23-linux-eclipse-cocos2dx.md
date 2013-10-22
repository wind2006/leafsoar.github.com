---
layout: post
title: Eclipse 组织跨平台开发 Cocos2d-x 游戏
date: 2013-4-23 16:15
comments: true
categories: Cocos2d-x
---

前面我们完成了，在 [Linux 上运行 cocos2d-x](/archives/2013/04-17-17.html) 游戏，然后使用 [Eclipse 开发 Linux 平台 cocos2d-x](/archives/2013/04-19-11.html) 游戏，最后完成了在 Linux 下使用 [Eclipse 开发 Android 平台的 cocos2d-x](/archives/2013/04-22-22.html) 游戏。

Ecilpse 开发 Linux 平台游戏使用的是 `proj.linux` 项目，Android 平台的使用的是 `proj.android` 项目，现在我们将所有项目整合，提供一个 Eclipse 完全开发 cocos2d-x 跨平台游戏的解决方案。我们将以上篇博文所建立的 Android 平台项目 `HelloWorld` 为基础，整合进 Linux 平台项目，并使用其作为开发调试的主要项目。

## Eclipse 组织说明

使用 Eclipse 组织 cocos2d-x 项目，除了 Eclipse 所管理的 proj.linux 项目和 proj.android 项目，我们还需要一个项目，用户管理项目根目录，至于为何如此，后面将会说明，最终项目如下所示：

<!-- more -->

``` bash
	HelloWorld/   		# Eclipse 项目，项目名：HelloWorld
	├── Classes
	│   ├── AppDelegate.cpp
	│   ├── AppDelegate.h
	│   ├── HelloWorldScene.cpp
	│   └── HelloWorldScene.h
	├── proj.android		# Android 项目，项目名：HelloWorld_j
	│   ├── AndroidManifest.xml
	│   ├── ant.properties
	│   ├── assets
	│   ├── bin
	│   ├── build_native.sh
	│   ├── build.xml
	│   ├── Classes
	│   ├── gen
	│   ├── jni
	│   ├── libs
	│   ├── local.properties
	│   ├── ndkgdb.sh
	│   ├── obj
	│   ├── proguard-project.txt
	│   ├── proj.android
	│   ├── project.properties
	│   ├── res
	│   ├── Resources
	│   └── src
	├── proj.linux		# Linux 项目，项目名：HelloWorld_dx
	│   ├── bin
	│   ├── Debug
	│   ├── main.cpp
	│   ├── main.h
	│   ├── main.o
	│   └── Makefile
	└── Resources
	    ├── CloseNormal.png
	    ├── CloseSelected.png
	    └── HelloWorld.png
```		

最终效果，一个 HelloWorld 目录，包含四个文件夹 Classes、proj.android、proj.linux 和 Resources，还有三个项目，但明眼所见只有两个项目，何来三个，除了 Android 平台，Linuxe 平台的开发项目，还有一个组织全代码工程的项目 **HelloWorld**，最终在 Eclipse 将显示三个项目，HelloWorld、HelloWorld_dx 和 HelloWorld_j,由于在 Eclipse 中同一个工作空间之中不能出现相同的项目名称，所以在 Linux 平台加了个 **_dx** 后缀，Android 平台加了个 **_j** 后缀，加以区分，后缀名称为何 不重要，只要区分项目名称就行，以保证能在 Eclipse 同时打开这三个项目。

Eclipse 打开 proj.linux 是为了开发 cocos2d-x 游戏源代码，调试运行之用，proj.android 是为了编译到 android 平台运行之用，为什么还要加一个 HelloWorld 总项目呢，我们看到 Resources 和 Classes 目录所在的位置很是特殊，作为 proj.linux 和 proj.android 项目，我们可以将这两个目录的源代码添加到编译路径，但从目录结构上来说，是属于它们的父目录，如果我们开发过程使用如 SVN 这样的代码管理工具，那么 proj.android 和 proj.linux 项目，将**照顾**不到这两个目录，修改文件，不能签入，不能更新，所以添加了 HelloWorld 项目，这样就可以在 Eclipse 编写调试运行，并且通过 HelloWorld 项目和 SVN 做整个工程的同步，这一点在团队合作开发尤为重要，即便是个人开发，也最好用代码管理跟踪记录我们的代码，这是一个很好的习惯。

## Eclipse 组织实现步骤

* 在[上篇博文](/archives/2013/04-22-22.html)，我们已经通过 Eclipse 添加了一个名为 HelloWorld 的 Android 平台项目。基于此，我们首先修改这个项目的名称 `HelloWorld` 改为 `HelloWorld_j`，只要右击项目 **Rename** 即可。
* 然后我们 Eclipse 新建项目 `New -> Other -> General -> Project -> Next` ,输入项目名称 `HelloWorld` 后 `Finish`。注意这里的项目类型是 **General**，并不是 Android 或是其它项目，因为我们只需要用此管理目录即可，由于工作空间中已经存在 HelloWorld 目录，会自动以此目录创建 HelloWorld 项目，创建完成便能够在  Eclipse HelloWorld 项目中看到三个文件夹。Classes、proj.android 和 Resources。
* 最后我们需要一个 proj.linux 项目，cocos2d-x 本身并没有明确的创建方式，不像 VS 和 Xcode  都有项目模板可以和 IDE 自动集成，不过我们可以 copy cocos2dx 自带的例子 [HelloCpp](/archives/2013/04-19-11.html) 中的 proj.linux 拿过来一用。将其目录的 proj.linux 复制到 HelloWorld 目录之下。
* 复制了 proj.linux 目录，我们用 Eclipse 导入此项目。导入过以后修改项目名 HelloCpp 为 HelloWorld_dx。

至此，在 Eclipse 已经有了三个项目 HelloWorld、HelloWorld_dx 和 HelloWorld_j 了，android 项目已经是配置好的，可以直接使用而 Linux 项目由于是从 HelloCpp copy 过来的工程文件，所以还需要做些配置修改，才能运行。

* 我们先要做到能通过 make 命令编译此项目，然后在配置使用 Eclipse 可以开发，并提供自动补全功能，CDT 的功能还是很强大很好用的。
* 参照 [此篇博文](/archives/2013/04-19-11.html) 修改 makefile 已使 make 能进行项目的编译运行。

如果出现如下错误：

``` bash
	../Classes/HelloWorldScene.cpp:2:31: fatal error: SimpleAudioEngine.h: No such file or directory
	compilation terminated.
	make: *** [../Classes/HelloWorldScene.o] Error 1
```	

则在 **Makefile** 文件中的 **INCLUDES**  添加 **CocosDenshion** 的相关内容，最终如下：

``` bash
	INCLUDES =  -I../ \
				-I../Classes \
				-I$(COCOS2DX_PATH) \
				-I$(COCOS2DX_PATH)/platform/third_party/linux \
				-I$(COCOS2DX_PATH)/platform/third_party/linux/libfreetype2 \
				-I$(COCOS2DX_PATH)/cocoa \
				-I$(COCOS2DX_PATH)/include \
				-I$(COCOS2DX_PATH)/platform \
				-I$(COCOS2DX_PATH)/platform/linux \
				-I$(COCOS2DX_PATH)/platform/third_party/linux/glew-1.7.0/glew-1.7.0/include/ \
				-I$(COCOS2DX_PATH)/kazmath/include \
				-I$(COCOS2DX_PATH)/platform/third_party/linux/libxml2 \
				-I$(COCOS2DX_PATH)/platform/third_party/linux/libjpeg  \
				-I$(COCOS2DX_ROOT)/CocosDenshion/include \			# 添加的内容
```				

此时 make 已经能够在 bin 目录之下生成可执行文件。然后我们在来配置 Eclipse 开发环境。

##Eclipse Linux 项目配置全攻略

我们虽然已经能够通过 make 编译，但是并不能通过 Ecilpse 编译，因为 proj.linux 是从 cocos2d-x 里面 copy 出来的，里面的配置多使用相对路径，这对项目来说显然是不可行的，所以需要修改成适用于任何位置的项目，让其只依赖 环境变量：

* 修改包含源代码目录，使之正确。此时 HelloWorld_dx 的 Classes 并不能正确显示源代码（视具体环境而定，如果 Classes 能看到源代码，跳过此步骤）。修改项目属性找到如下选项 `C/C++ General -> Paths and Symbols` ，定位到 `Source Location`  选项卡，**delete** `/HelloWorld_dx/Classes`  项，然后添加一个 **Link Folder** ，` Link folder in the file system` 勾上，填写值为 **PARENT-1-PROJECT_LOC/Classes**，Folder name 自动变为 Classes 后点击 **OK**。 此时便能够在 HelloWorld_dx 看到 Classes 文件夹和其中的源代码了。
* 修改包含头文件路径，使之正确。我们点开项目的 Includes 看到所引用的相关 cocos2dx 头文件路径都不正确，还是定位到 `C/C++ General -> Paths and Symbols` 选项查看 **Include** 选项卡中 **GNU C++**  右侧包含很多头文件，修改成 **${cocos2dx_loc}/cocos2dx** 诸如此值，其它错误路径同样修改。
* 我们需要在 `Eclipse -> Windows ` 属性里面 `C/C++ -> Build -> Build Variable` 添加 **cocos2dx_loc** 的变量，其变量值为 [cocos2dx-path] 主目录。
* 修改项目类型为是用自定义 Makefile 编译运行。

我们就能完全用 Eclipse 编写 cocos2d-x 程序了，并且有着完善的代码提示功能。提高开发效率。从代码的编写运行调试，到跨平台的 Android 运行，Eclipse 组织项目可以同步 SVN 多人协作。到此，已经提供了一个基本的解决方案 ~

当然开发环境的灵活性，也许可能遇到我们未知的问题，总是在不断的发现问题，并解决问题。灵活的部署，项目的组织，环境的配置。有一个好用顺手的开发环境是我们在开发前必做的准备 ~
