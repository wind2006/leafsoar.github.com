---
layout: post
title: Eclipse Cocos2d-x 开发自动管理
date: 2013-4-24 12:15
comments: true
categories: Cocos2d-x
---

## Makefile Android.mk 引发的思索

在我们编写 Android 平台 cocos2d-x 游戏的时候，我们除了编写 `Classes` 之内的源代码文件之外，我们还需要维护其编译文件 Android.mk，如我们在 Classes 添加新的源文件，那么我们就要在 Android.mk 配置添加其编译路径，如：

``` bash
	LOCAL_SRC_FILES := hellocpp/main.cpp \
	                   ../../Classes/AppDelegate.cpp \
	                   ../../Classes/HelloWorldScene.cpp
```					   

每添加一个源文件，我们就要手动添加一个配置，始其能够被编译，同样的，在 proj.linux 的 Makefile 文件也有这样的情况：

``` makefile
	OBJECTS = ./main.o \
	        ../Classes/AppDelegate.o \
	        ../Classes/HelloWorldScene.o
```
			
当然让我们手动维护其配置，当然可以，不过麻烦非常，对于像我这样“懒惰”之人，当然需要想办法让其自动管理喽 ~

<!-- more -->

## 自动编译、自动维护

如果要自动维护编译文件之内的源代码文件，我们需要的无非就是所有的源代码文件及其路径，而这样的工作可以通过 Linux 强大的命令 find 来实现自动完成，Android.mk 文件如下([获取源码](https://github.com/leafsoar/ls-cocos2d-x/blob/master/HelloWorld/proj.android/jni/Android.mk))：

``` makefile
	LOCAL_PATH := $(call my-dir)
	
	include $(CLEAR_VARS)
	
	LOCAL_MODULE := game_shared
	
	LOCAL_MODULE_FILENAME := libgame

	# 定义 all-cpp-files 返回当前路径和 Classes 路径想的所有 cpp 文件，注意：这里只考虑 cpp 而没有 c，如果需要自行添加
	define all-cpp-files
	$(patsubst jni/%,%, $(shell find $(LOCAL_PATH)/../../Classes/ $(LOCAL_PATH)/hellocpp -name "*.cpp"))  
	endef

	# 这里使用新的方式替换换来的方式，以自动添加源文件
	LOCAL_SRC_FILES := $(call all-cpp-files)

	#LOCAL_SRC_FILES := hellocpp/main.cpp \
	#                  ../../Classes/AppDelegate.cpp \
	#                  ../../Classes/HelloWorldScene.cpp
		                   
	LOCAL_C_INCLUDES := $(LOCAL_PATH)/../../Classes 
	
	LOCAL_WHOLE_STATIC_LIBRARIES := cocos2dx_static cocosdenshion_static cocos_extension_static
	            
	include $(BUILD_SHARED_LIBRARY)
	
	$(call import-module,CocosDenshion/android) \
	$(call import-module,cocos2dx) \
	$(call import-module,extensions)		# 根据自己需要是否启用，上面的静态库同样
```	

这样一个 Android.mk 算是**万能**的配置了，基本能满足我们编写 cocos2d-x 游戏的大多数需求了，当然如果你使用了第三方库，当然还是需要手动添加一下配置了，不过就源文件来说，不需要手动维护，倒是省事许多。下面在贴一个 Makefile 的万能配置([获取源码](https://github.com/leafsoar/ls-cocos2d-x/blob/master/HelloWorld/proj.linux/Makefile))：

``` makefile
	CC      = gcc
	CXX     = g++
	TARGET	= leafsoar			# 为了保持通用性，干脆起个不相干的目标文件，此名随意
	CCFLAGS = -Wall
	CXXFLAGS = -Wall
	VISIBILITY = 
	
	# COCOS2DX_ROOT = /home/leafsoar/...		# 如果已经配置过此环境变量，可以不需要此，否则添加此变量值
	
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
				-I$(COCOS2DX_ROOT)/CocosDenshion/include \
	#			-I$(COCOS2DX_ROOT)/extensions/ \			# 根据自己需要是否包含 extensions 扩展
	
	DEFINES = -DLINUX

	# 获取源文件列表
	define all-cpp-files
	$(patsubst ./%,%, $(shell find  ../Classes ./ -name "*.cpp"))  
	endef

	# 我是打算让所以编译后的 ".o" 临时文件，全部生成在 "obj" 目录，而不是和源代码同目录
	define all-cpp-dir
	$(patsubst ../%,obj/%, $(shell find  ../Classes -type d))  
	endef

	# obj 默认目录
	OBJDIR=obj/Classes

	# 获取所有的编译文件列表
	OBJECTS=$(patsubst %.cpp,$(OBJDIR)/%.o,$(call all-cpp-files))
	
	# 获取所有的编译文件路径，如果不存在路径则，编译可能出现问题
	OBJECTS_DIR=$(call all-cpp-dir)
	
	# 如果目录不存在，则创建相应的目录，-p 命令保证了，如果存在，不需要重新创建，这样没有修改的源文件就无需重新编译，提高速度
	$(shell mkdir -p obj)
	$(shell mkdir -p $(OBJECTS_DIR))
	
	#echo:
	#	@echo $(OBJECTS_DIR)
	
	#OBJECTS = ./main.o \
	#        ../Classes/AppDelegate.o
	
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
	SHAREDLIBS += -Wl,-rpath,$(SHAREDLIBS_DIR)
	SHAREDLIBS += -L$(COCOS2DX_ROOT)/cocos2dx/platform/third_party/linux/glew-1.7.0/glew-1.7.0/lib -lGLEW
	SHAREDLIBS += -Wl,-rpath,$(COCOS2DX_ROOT)/cocos2dx/platform/third_party/linux/glew-1.7.0/glew-1.7.0/lib
	
	
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
	release: SHAREDLIBS += -L.$(COCOS2DX_ROOT)/lib/linux/Release -lcocos2d -lrt -lz -lcocosdenshion
	release: SHAREDLIBS += -Wl,-rpath,$(COCOS2DX_ROOT)/lib/linux/Release/
	release: DEFINES += -DNDEBUG
	release: $(TARGET)
	
	####### Build rules
	$(TARGET): $(OBJECTS) 
		mkdir -p $(BIN_DIR)
		$(CXX) $(CXXFLAGS) $(INCLUDES) $(DEFINES) $(OBJECTS) -o $(BIN_DIR)/$(TARGET) $(SHAREDLIBS) $(STATICLIBS)
	
	####### Compile
	$(OBJDIR)/%.o: %.cpp
		$(CXX) $(CXXFLAGS) $(INCLUDES) $(DEFINES) $(VISIBILITY) -c $< -o $@
	
	%.o: %.c
		$(CC) $(CCFLAGS) $(INCLUDES) $(DEFINES) $(VISIBILITY) -c $< -o $@
	
	clean: 
		rm -f $(OBJECTS) $(TARGET) core
```		
	
有了此 Makefile 我们就能满足我们绝大多数需求了，并且还做了目录优化，将所有源文件生成的 `.o ` 文件统一放在了 obj 目录之下，方便管理，否则源文件路径会稍显零乱。实现方式，就是通过命令先创建符合条件的路径，然后修改其编译生成的临时文件路径。这只是我在使用 cocos2d-x 2.0.4 才出现的问题，而在最新版本2.1.12好似做了些修改，不需要显示的修改其 `.o` 文件路径。

## 获取项目

关于如此管理项目，我在网上提供的完整的例子，可以从 [GitHub](https://github.com/leafsoar/ls-cocos2d-x/tree/master/HelloWorld) [^1] 上面下载，包含了一个完整 HelloWorld 工程项目。可以从这理获取，其中 [Android.mk](https://github.com/leafsoar/ls-cocos2d-x/blob/master/HelloWorld/proj.android/jni/Android.mk) 和 [Makefile](https://github.com/leafsoar/ls-cocos2d-x/blob/master/HelloWorld/proj.linux/Makefile) 文件可以直接使用。

如果你是在 Windows 下使用 cygwin 编译，那么这篇文章只能作为参考，其也是 unix 环境的一个模拟，但这里并不能确定其过程是否会出现什么问题。

[^1]:  关于本博客以后可能会出现的例子代码，都将放在 [GitHub](https://github.com/leafsoar/ls-cocos2d-x) 之上，可以从 [这里](https://github.com/leafsoar/ls-cocos2d-x) 获取到所有的内容。
