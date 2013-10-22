---
layout: post
title: 善其事先利其器 Eclipse 开发 Cocos2d-x
date: 2013-4-19 11:20
comments: true
categories: Cocos2d-x
---

[上篇博客](/archives/2013/04-17-17.html)，我们实现了在 Linux 下运行 cocos2d-x 游戏，这意味着我们可以在 Linux PC 机上进行游戏的编写、运行和调试，提高开发效率，而在手机上运行的过程大致分为：

* PC 上编写游戏源代码
* NDK 编译生成 so 库文件
* Android 工程打包生成 Apk 文件
* 传输到模拟器或真机运行

这么个手机上运行的过程繁琐非常，而且这只是运行在 Android 上的过程，如果通过 Android Jni 调用调试 C++ 更为麻烦，现今的 NDK 版本应该支持 Android Jni C++ 程序调试了。而在 PC 端开发运行和调试，**偶尔** 在手机上运行看看效果的开发过程，更为合适！虽然 cocos2d-x 是跨平台的，但常在目标平台运行，可以提前发现因平台特性可能出现的问题，并即时解决。

<!-- more -->

## Eclipse 上开发 Cocos2d-x  游戏

善其事先利其器，在做开发之前，搭建一个顺手的开发环境是必要的准备！

### 安装开发环境

在 Debian  系统下，可以直接通过 `sudo apt-get install eclipse` 安装开发环境，并自动安装依赖项，如 JDK 等。而这里是从官方网站下载最新版本。手动配置，可以选择一个自己熟悉的方式。

* 这里使用的是 [**Eclipse Juno (4.2)**](http://www.eclipse.org/downloads/) 可以下载 **IDE for Java** 版本之后在安装 **CDT**，也可以直接下载 **IDE for C/C++** 版本的 Eclipse
* 安装 [**JDK**](http://www.oracle.com/technetwork/java/javase/downloads/index.html) ，只需要 JAR 支持，就能运行 Eclipse 编写 C++ 了，为了以后编写 Java Android 程序，这里直接安装 JDK。
* 下载之后配置环境变量，把 JDK bin 目录添加到 $PATH 环境变量。

### 第一个程序 **HelloCpp**

现在的准备环境 Eclipse 和已经配置好的 CDT，并且能够运行普通 C++ 项目，而 cocos2d-x 的 Linux 也就是一个普通的 C++ 项目（ cocos2d-x 的跨平台特性）。我们先来看一下 cocos2d-x 是如何组织项目的，如下所示：

``` bash
	HelloCpp/
	├── Classes		# cocos2d-x 游戏源代码
	├── proj.android		# Android 平台项目目录组织
	├── proj.blackberry		# blackberry 项目组织
	├── proj.ios		# iOS 平台项目
	├── proj.linux		# Linux PC 项目，也是我们要用到的
	├── proj.mac		# Mac 项目
	├── proj.win32		# Windows VS 下 Win32项目
	└── Resources		# 游戏资源，包含图片、声音、字体等
```	

在此处，我们只需要用到 `Classes`、`Resources` 和 `proj.linux` 就可以了，其中`proj.[平台]` 就是不同的开发平台，之后编译运行在不同平台的项目组织，而他们共用了源代码 `Classes` 和 `Resources` 资源文件。如果我们今后自己创建项目，最好也保持这样的组织方式！

用 Eclipse 打开添加 HelloCpp 项目到：

Eclipse -> File -> Import -> General -> Existing Projects into Workspace -> Browse 浏览目录，选中 `[cocos2dx-path]/samples/HelloCpp/proj.linux`， 点击 **OK** 之后 **Finish** ，我们就能在 Eclipse 看见被导入的项目了。

现在我们试着运行 HelloCpp 项目，右击项目 **Build Project**（如果此项“灰显”不能点击的话，打开 Eclipse Problems 窗口，把 Errors 和 Warnings 都给删除即可），此时编译会编译不过去，提示如下错误信息：

``` bash
	Description	Resource	Path	Location	Type
		cannot find -lcocosdenshion	HelloCpp		 	C/C++ Problem
		cannot find -lcocos2d	HelloCpp		 	C/C++ Problem
		make: *** [HelloCpp] Error 1	HelloCpp		 	C/C++ Problem
```		

在继续操作之前，我们先来了解一下 Eclipse 中 编写 C/C++ 程序的一般方式，作为编写程序来说，我们只要包含正确头文件的位置，就能够使用 CDT 带来的快捷，自动补全，而作为编译过程，这里有两种方式。**其一： Eclipse 组织编译** , Eclipse 之中设置很多参数，源代码路径，包含文件，包含的库文件等以系列信息，根据这些信息，它就能为我们自动生成 makefile 文件（具体见项目之下 `Debug` 目录），然后再根据这个文件自动编译，这样我们只需要写代码程序，不需要手动维护 Makefile 文件。**其二：自定义 Makefile**，如 HelloCpp 项目之中包含一 Makefile 文件，我们只要进入这个目录执行 `make` 命令，就能完成项目的编译，上篇博客就是使用这种方式，而在 Eclipse 自动调用 make 效果同样。

而编写 **cocos2d-x** 游戏，这里我推荐第二种方式，有几点原因和优势。如果使用 Eclipse 自动生成的 makefile 编译方式，那么在编译当前项目之时，我们需要在 Eclipse 编译它的依赖项目，这会始事情变的复杂，其依赖项目有两个是必须的(如 [`[cocos2dx-path]/cocos2dx` 和 `[cocos2dx-path]/CocosDenshion`)，还有可选的(如`[cocos2dx-path]/external/Box2D` 物理引擎库)。还有优势，Eclipse 只作为项目组织编写，如果没有 Eclipse 环境，我们同样能够用 make 编译整个项目，所以维护好我们的 Makefile 可以省事很多。

我们编译出现的错误，找不到库是因为 Eclipse 自动生成的 makefile 不知道从哪里去找库文件，当然我们可以把库项目添加到 Eclipse 之中，然后先把库项目编译成功，再来编译此项目，当然是完全可行的，但不推荐。

知道原因后，我们做简单的修改

### 修改项目，编译运行

* 右击项目，查看属性 (Properties)
* 选中 `C/C++ Build` 项，找到 `Makefile generation` 组合框之中的复选框 `Generate Makefiles automatically`，将 **勾** 去掉，以使用我们自己编写的 Makefile 编译。
* 同样 `C/C++ Build` 项，找到标签 `Behaviour` 将 `Build (Incremental build)` 复选框之后的 变量值 **all** 删除，设置为空，否则编译项目会出现 `make: *** No rule to make target 'all'.  Stop.	HelloCpp` 的 错误

此时我们在 **Build Project** 项目，就已经在 `bin` 目录生成可执行文件了。我们在项目右击 `Run as` -> `Local C/C++ Application` 但却运行不了，与 [上篇博客](/archives/2013/04-17-17.html) 相比，虽然生成的方式略有不同，但本质一样，都是通过 **Makefile** 生成，由于配置原因，必须要在 **当前目录之下才能运行** 。这是因为 Makefile 之中默认配置的库路径问题所致。

Makefile 使用的是 **相对路径** 引用库文件的，这也就是因为只有在 当前目录运行有效(工作目录在当前目录)，而在 Eclipse 之中运行无效的原因，Eclipse 的运行 **工作目录** 不是可执行程序的目录！！！

不能运行，当然也就不能调试，我们还需要继续配置。

### 修改配置，集成开发环境并运行调试

现在开始到维护我们的 Makefile 文件了，修改 Makefile 之前，我们先定一个环境变量， **$COCOS2DX_ROOT** 以标示 cocos2dx的主目录。在 Eclipse 之中操作：

``` bash
	Ecilpse -> Windows -> Preferences -> C/C++ -> Build -> Environment
```	

点击 **Add** ， **Name** 为 **COCOS2DX_ROOT**，**Value** 为 **[cocos2dx-path]**

(此处视你的环境而定，比如 **"[xxx/xxx/xxx]/cocos2d-2.0-x-2.0.4"**)

然后修改我们的 Makefile 文件，把一些相对路径转换成基于环境变量的绝对路径，最终如下：

``` makefile
	CC      = gcc
	CXX     = g++
	TARGET	= HelloCpp
	CCFLAGS = -Wall
	CXXFLAGS = -Wall
	VISIBILITY =
	 
	COCOS2DX_PATH = $(COCOS2DX_ROOT)/cocos2dx
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
	
	
	DEFINES = -DLINUX
	
	OBJECTS = ./main.o \
	        ../Classes/AppDelegate.o \
	        ../Classes/HelloWorldScene.o 
	
	LBITS := $(shell getconf LONG_BIT)
	ifeq ($(LBITS),64)
	STATICLIBS_DIR = $(COCOS2DX_ROOT)/cocos2dx/platform/third_party/linux/libraries/lib64
	else
	STATICLIBS_DIR = $(COCOS2DX_ROOT)/cocos2dx/platform/third_party/linux/libraries
	endif
	STATICLIBS = 
	STATICLIBS = $(STATICLIBS_DIR)/libfreetype.a \
					$(STATICLIBS_DIR)/libxml2.a \
					$(STATICLIBS_DIR)/libpng.a \
					$(STATICLIBS_DIR)/libjpeg.a \
					$(STATICLIBS_DIR)/libtiff.a \
	#				$(STATICLIBS_DIR)/libGLEW.a \
	
	SHAREDLIBS = 
	ifeq ($(LBITS),64)
	SHAREDLIBS_DIR = $(COCOS2DX_ROOT)/CocosDenshion/third_party/fmod/lib64/api/lib
	SHAREDLIBS = -L$(SHAREDLIBS_DIR) -lfmodex64
	else
	SHAREDLIBS_DIR = $(COCOS2DX_ROOT)/CocosDenshion/third_party/fmod/api/lib
	SHAREDLIBS = -L$(SHAREDLIBS_DIR) -lfmodex
	endif
	
	SHAREDLIBS += -lglfw -lGL
	#SHAREDLIBS += -L../../../lib/linux/Debug -lcocos2d -lrt -lz -lcocosdenshion -Wl,-rpath,../../../../lib/linux/Debug/ 
	SHAREDLIBS += -Wl,-rpath,$(SHAREDLIBS_DIR)
	#SHAREDLIBS += -Wl,-rpath,../../../../cocos2dx/platform/third_party/linux/glew-1.7.0/glew-1.7.0/lib
	SHAREDLIBS += -L$(COCOS2DX_ROOT)/cocos2dx/platform/third_party/linux/glew-1.7.0/glew-1.7.0/lib -lGLEW
	SHAREDLIBS += -Wl,-rpath,$(COCOS2DX_ROOT)/cocos2dx/platform/third_party/linux/glew-1.7.0/glew-1.7.0/lib
	
	#$(shell ../../build-linux.sh $<)
	
	BIN_DIR_ROOT=bin
	BIN_DIR = $(BIN_DIR_ROOT)
	
	debug: BIN_DIR = $(BIN_DIR_ROOT)/debug
	debug: CCFLAGS += -g3 -O0
	debug: CXXFLAGS += -g3 -O0
	debug: SHAREDLIBS += -L$(COCOS2DX_ROOT)/lib/linux/Debug -lcocos2d -lrt -lz -lcocosdenshion
	debug: SHAREDLIBS += -Wl,-rpath,$(COCOS2DX_ROOT)/lib/linux/Debug/
	debug: DEFINES += -DDEBUG
	debug: $(TARGET)
	
	release: BIN_DIR = $(BIN_DIR_ROOT)/release
	release: CCFLAGS += -O3
	release: CXXFLAGS += -O3
	release: SHAREDLIBS += -L$(COCOS2DX_ROOT)/lib/linux/Release -lcocos2d -lrt -lz -lcocosdenshion
	release: SHAREDLIBS += -Wl,-rpath,$(COCOS2DX_ROOT)/lib/linux/Release/
	release: DEFINES += -DNDEBUG
	release: $(TARGET)
	
	####### Build rules
	$(TARGET): $(OBJECTS) 
		mkdir -p $(BIN_DIR)
		$(CXX) $(CXXFLAGS) $(INCLUDES) $(DEFINES) $(OBJECTS) -o $(BIN_DIR)/$(TARGET) $(SHAREDLIBS) $(STATICLIBS)
	
	####### Compile
	%.o: %.cpp
		$(CXX) $(CXXFLAGS) $(INCLUDES) $(DEFINES) $(VISIBILITY) -c $< -o $@
	
	%.o: %.c
		$(CC) $(CCFLAGS) $(INCLUDES) $(DEFINES) $(VISIBILITY) -c $< -o $@
	
	clean: 
		rm -f $(OBJECTS) $(TARGET) core
```		
	

需要注意 **$(COCOS2DX_ROOT)** 和 **$(COCOS2DX_PATH)**  的区别，前者是 cocos2d-x 总目录，后者是 cocos2d-x 其中基础库的目录。

如果需要在命令行之内直接 `make` 编译，只要创建环境变量 `COCOS2DX_ROOT` 即可。

修改之后，我们再次编译，运行，就能够在 Eclipse 之中运行游戏，看其效果，并且可以在代码之中设置断点，**调试运行** 。


### 稍作总结

在 Linux 下使用 Eclipse 开发 cocos2d-x 游戏的环境配置大致是这样的了，Eclipse 作为编写代码，自己使用 Makefile 来维护项目的编译的推荐的方式。这只是实现了基本的配置方法，实际问题远不止这些，比如 此环境是默认的，在 Eclipse 项目中引用的库文件路径还是 **相对路径**，如果移动到其它目录(项目肯定用我们自己的目录)，则破坏了 Eclipse 的开发环境。当然最终的编写是为了运行在手机之中，以后也会给出怎么编译到手机 Android 平台，并给出相应的推荐做法，以简化开发步骤！
