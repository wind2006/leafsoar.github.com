---
layout: post
title: Cocos2d-x 程序是如何开始运行与结束的
date: 2013-05-5 23:30
comments: true
categories: Cocos2d-x
---

**题记：对于技术，我们大可不必挖得那么深，但一定要具备可以挖得很深的能力**

## 问题的由来
怎么样使用 Cocos2d-x 快速开发游戏，方法很简单，你可以看看其自带的例程，或者从网上搜索教程，运行起第一个 Scene **HelloWorldScene**，然后在 HelloWorldScene 里面写相关逻辑代码，添加我们的层、精灵等 ~ 我们并不一定需要知道 Cocos2d-x 是如何运行或者在各种平台之上运行，也不用知道 Cocos2d-x 的游戏是如何运行起来的，它又是如何渲染界面的 ~~~

我们只用知道 Cocos2d-x 的程序是由 **AppDelegate** 的方法 **applicationDidFinishLaunching** 开始，在其中做些必要的初始化，并创建运行**第一个 CCScene** 即可，正如我们第一次使用各种编程语言写 **Hello World!** 的程序一样，如 Python 打印：

``` python
	print('Hello World!')
```
<!-- more -->

我们可以不用关心其是怎么实现的，我们只要知道这样就能打印一句话就够了，这就是 **封装所带来的好处** 。

Cocos2d-x 自带的例程已经足够丰富，但是有些问题并不是看看例子，调用其方法就能明白的事情，在这里一叶遇到了如下问题：

``` c++
	// AppDelegate.cpp 文件
	
	AppDelegate::AppDelegate()
	{
		CCLog("AppDelegate()");		// AppDelegate 构造函数打印
	}
	
	AppDelegate::~AppDelegate()
	{
		CCLog("AppDelegate().~()");		// AppDelegate 析构函数打印
	}

	// 程序入口
	bool AppDelegate::applicationDidFinishLaunching()
	{
		// initialize director
		CCDirector *pDirector = CCDirector::sharedDirector();
		pDirector->setOpenGLView(CCEGLView::sharedOpenGLView());
	
		// 初始化，资源适配，屏幕适配，运行第一个场景等代码
		...
		...
		...
		
		return true;
	}
	
	void AppDelegate::applicationDidEnterBackground()
	{
		CCDirector::sharedDirector()->pause();
	}
	
	void AppDelegate::applicationWillEnterForeground()
	{
		CCDirector::sharedDirector()->resume();
	}
```	

此时我并不知道程序运行时，何时调用 AppDelegate 的构造函数，析构函数和程序入口函数，我们只要知道，程序在这里调用了其构造函数，然后进入入口函数执行其过程，最后再调用其析构函数即可。然而**事与愿违**，在实际执行的过程中，发现程序只调用其构造函数和入口函数，而直到程序结束运行，都 **没有调用其析构函数**。要验证此说法很简单，只要如上在析构函数中调用打印日志便可验证。

发生这样的情况，让我 **在构造函数创建[资源]，并且在析构函数中释放[资源]** 的想法不能完成！！！ 我们**知道它是从哪里开始运行，但却不知道它在哪里结束！**疑问，唯有疑问！

	
## 两个入口

程序入口的概念是**相对的**，**AppDelegate** 作为跨平台程序入口，在这之上做了另一层的封装，封装了不同平台的不同实现，比如我们通常认为一个程序是由 **main** 函数开始运行，那我们就去找寻，我们看到了在 **proj.linux** 目录下存在 **main.cpp** 文件，这就是我们要看的内容，如下：

``` c++
	#include "main.h"
	#include "../Classes/AppDelegate.h"
	#include "cocos2d.h"
	
	
	#include <stdlib.h>
	#include <stdio.h>
	#include <unistd.h>
	#include <string>
	
	USING_NS_CC;
	
	// 500 is enough?
	#define MAXPATHLEN 500
	
	int main(int argc, char **argv)
	{
	    // get application path
	    int length;
	    char fullpath[MAXPATHLEN];
	    length = readlink("/proc/self/exe", fullpath, sizeof(fullpath));
	    fullpath[length] = '\0';
	
	    std::string resourcePath = fullpath;
	    resourcePath = resourcePath.substr(0, resourcePath.find_last_of("/"));
	    resourcePath += "/../../../Resources/";
	    
	    // create the application instance
	    AppDelegate app;
	    CCApplication::sharedApplication()->setResourceRootPath(resourcePath.c_str());
	    CCEGLView* eglView = CCEGLView::sharedOpenGLView();
	    eglView->setFrameSize(720, 480);
	//    eglView->setFrameSize(480, 320);
	
	    return CCApplication::sharedApplication()->run();
	}
```	
	
在这里我们看见了程序的**真正**入口，包含一个 main 函数，从此进入，执行 cocos2d-x 程序。

我们看到 main 就知道其是入口函数，那么没有 main 函数就没有入口了吗？显然不是，以 Android 平台启动 cocos2d-x 程序为例。我们找到 Android 平台与上面 **等价** 的入口点，`proj.android/jni/hellocpp/main.cpp`：

``` c++
	#include "cocos2d.h"
	#include "AppDelegate.h"
	#include "platform/android/jni/JniHelper.h"
	#include <jni.h>
	#include <android/log.h>
	
	#define  LOG_TAG    "main"
	#define  LOGD(...)  __android_log_print(ANDROID_LOG_DEBUG,LOG_TAG,__VA_ARGS__)
	
	using namespace cocos2d;
	
	extern "C"
	{
	
	jint JNI_OnLoad(JavaVM *vm, void *reserved)
	{
	    JniHelper::setJavaVM(vm);
	
	    return JNI_VERSION_1_4;
	}
	
	void Java_org_cocos2dx_lib_Cocos2dxRenderer_nativeInit(JNIEnv*  env, jobject thiz, jint w, jint h)
	{
	    if (!CCDirector::sharedDirector()->getOpenGLView())
	    {
	        CCEGLView *view = CCEGLView::sharedOpenGLView();
	        view->setFrameSize(w, h);
	
	        AppDelegate *pAppDelegate = new AppDelegate();
	        CCApplication::sharedApplication()->run();
	    }
	    else
	    {
	        ccDrawInit();
	        ccGLInvalidateStateCache();
	        
	        CCShaderCache::sharedShaderCache()->reloadDefaultShaders();
	        CCTextureCache::reloadAllTextures();
	        CCNotificationCenter::sharedNotificationCenter()->postNotification(EVNET_COME_TO_FOREGROUND, NULL);
	        CCDirector::sharedDirector()->setGLDefaultValues(); 
	    }
	}
```	

我们并没有看到所谓的 main 函数，这是由于不同的平台封装所以有着不同的实现，在 Android 平台，默认是使用 Java 开发，可以使用 Java 通过 Jni 调用 C++ 程序，而这里也正式如此。我们暂且只需知道，由 Android 启动一个应用，通过各种**峰回路转**，最终执行到了 `Java_org_cocos2dx_lib_Cocos2dxRenderer_nativeInit` 函数，由此，变开始了我们 cocos2d-x Android 平台的程序入口处。对于跨平台的 cocos2d-x 来说，除非必要，否则可不必深究其理，比如想要使用 Android 平台固有的特性等，那就需要更多的了解 Jni 使用方法，以及 Android 操作系统的更多细节。

所以说程序的入口是相对的，正如博文开始的 `print('Hello World')` 一样，不同的语言，不同平台总有着不同的实现。

这里我们参考了两个不同平台的实现， Linux 和 Android 平台 cocos2d-x 程序入口 main.cpp的实现，那么其它平台呢，如 iOS ,Win32 等 ~~~ 殊途同归，其它平台程序的入口必然包含着其它平台的不同 **封装实现** ，知道有**等价**在此两平台的程序入口即可。而通过这两个平台也足够解决我们的疑问，**程序的开始与结束** ~

## 问题的推测

我们就从 Linux 和 Android 这两个平台的入口函数开始，看看 cocos2d-x 的执行流程到底为何？何以发生只执行了 AppDelegate 的构造函数，而没有析构函数。在查看 cocos2d-x 程序代码时，我们只关注 **必要的** 内容，何谓 必要，只要能解决我们此时的疑问即可！在两个平台的入口函数，我们看到如下内容：

``` c++
	// Linux 平台关键代码
	int main(int argc, char **argv)
	{
		// 初始化等内容
		...
		...
		// 创建 app 变量
	    AppDelegate app;	
		...
		...
		// 执行 核心 run() 方法
	    return CCApplication::sharedApplication()->run();
	}

	// Android 平台关键代码
	void Java_org_cocos2dx_lib_Cocos2dxRenderer_nativeInit(JNIEnv*  env, jobject thiz, jint w, jint h)
	{
	    if (!CCDirector::sharedDirector()->getOpenGLView())
	    {
	        CCEGLView *view = CCEGLView::sharedOpenGLView();
	        view->setFrameSize(w, h);

			// 创建 AppDelegate 对象
	        AppDelegate *pAppDelegate = new AppDelegate();
			// 执行 核心 run() 方法
	        CCApplication::sharedApplication()->run();
	    }
	    else
	    {
			...
			...
	    }
	}
```	

不同的平台，却实现相同操作，创建 **AppDelegate** 变量和执行 **run** 方法。下面将以 Linux 平台为例，来说明程序是如何开始与结束的，因为 Linux 的内部实现要简单一点，而 Android 平台的实现稍显麻烦，Jni 之间来回调用，对我们理解 cocos2d-x 的执行流程反而有所 **阻碍**，况且 cocos2d-x 本身就是跨平台的程序。不必拘泥于特有平台的专有特性。

## 程序的流程 （这里以 Linux 的实现为主，其它平台触类旁通即可）

### AppDelegate 与 CCApplication

我们从 **main.cpp** 中 **CCApplication::sharedApplication()->run();** 这一句看起，这一句标志着， cocos2d-x 程序正式开始运行，一点点开始分析，我们定位到 **sharedApplication()** 方法的实现，这里只给出 **必要** 的代码，具体看一自己直接看源码：

``` c++
	// [cocos2dx-path]/cocos2dx/platform/linux/CCApplication.cpp
	...
	// 此变量为定义了一个 CCApplication 的静态变量，也及时自己类型本身，实现单例模式
	CCApplication * CCApplication::sm_pSharedApplication = 0;
	...
	// 构造函数，将所创建的 对象直接付给其静态变量
	CCApplication::CCApplication()
	{
		// 断言在此决定着此构造函数只能运行一次
		CC_ASSERT(! sm_pSharedApplication);
		sm_pSharedApplication = this;
	}
	
	CCApplication::~CCApplication()
	{
		CC_ASSERT(this == sm_pSharedApplication);
		sm_pSharedApplication = NULL;
		m_nAnimationInterval = 1.0f/60.0f*1000.0f;
	}

	// run 方法，整个 cocos2d-x 的主循环在这里开始
	int CCApplication::run()
	{
		// 首次启动调用初始化函数
		if (! applicationDidFinishLaunching())
		{
			return 0;
		}

		// 游戏主循环，这里 Linux 的实现相比其它平台的实现，简单明了
		for (;;) {
			long iLastTime = getCurrentMillSecond();
			// 在循环之内调用每一帧的逻辑，组织并且控制 cocos2d-x 之中各个组件
			CCDirector::sharedDirector()->mainLoop();
			long iCurTime = getCurrentMillSecond();
			// 这里的几个时间变量，可以控制每一帧所运行的 最小 时间，从而控制游戏的帧率
			if (iCurTime-iLastTime<m_nAnimationInterval){
				usleep((m_nAnimationInterval - iCurTime+iLastTime)*1000);
			}
	
		}
		// 注意，这里的 for 循环，并没有退出循环条件，这也决定着 run() 方法永远也不会返回
		return -1;
	}
	
	// 方法直接返回了静态对象，并且做了断言，也既是在调用此方法之前，
	// 必须事先创建一个 CCApplication 的对象，以保证其静态变量能够初始化，否则返回空
	CCApplication* CCApplication::sharedApplication()
	{
		CC_ASSERT(sm_pSharedApplication);
		return sm_pSharedApplication;
	}
```	

从上面的内容可以看出，从 **sharedApplication()** 方法，到 **run()** 方法，在这之前，我们需要调用到它的构造函数，否则不能运行，这就是为什么在 **CCApplication::sharedApplication()->run();** 之前，我们首先使用了 **AppDelegate app;** 创建 AppDelegate 变量的原因！ 嗯 ！！ **AppDelegate 和 CCAppliation 是什么关系！** 由 AppDelegate 的定义我们可以知道，它是 CCApplication 的子类，在创建子类对象的时候，调用其构造函数的同时，父类构造函数也会执行，然后就将 AppDelegate 的对象赋给了 CCApplication 的静态变量，而在 AppDelegate 之中我们实现了 **applicationDidFinishLaunching** 方法，所以在 CCApplication 中 **run** 方法的开始处调用的就是 AppDelegate 之中的实现。而我们在此方法中我们初始化了一些变量，创建了第一个 CCScene 场景等，之后的控制权，便全权交给了 **CCDirector::sharedDirector()->mainLoop();** 方法了。

（这里的实现机制，不做详细说明，简单说来：**applicationDidFinishLaunching** 是由 CCApplicationProtocol 定义，CCApplication 继承， AppDelegate 实现的 ~）

**比较重要的所在，for 循环并没有循环退出条件，所以 run 方法永远不会返回。那么是怎么结束的呢！要学会存疑！**

### 从 CCApplication 到 CCDirector

cocos2d-x 程序已经运行起来了，我们继续下一步，**mainLoop** 函数：

``` c++
	// [cocos2dx-path]/cocos2dx/CCDirector.cpp
	...
	// 定义静态变量，实现单例模式
	static CCDisplayLinkDirector *s_SharedDirector = NULL;
	...
	// 返回 CCDirector 实例
	CCDirector* CCDirector::sharedDirector(void)
	{
		// 判断静态变量，以保证只有一个实例
	    if (!s_SharedDirector)
	    {
	        s_SharedDirector = new CCDisplayLinkDirector();
	        s_SharedDirector->init();
	    }
		// CCDisplayLinkDirector 为 CCDirector 的子类，这里返回了其子类
	    return s_SharedDirector;
	}

	// mainLoop 方法的具体实现
	void CCDisplayLinkDirector::mainLoop(void)
	{
		// 此变量是我们需要关注，并且跟踪的，因为它决定着程序的结束时机
	   if (m_bPurgeDirecotorInNextLoop)
	   {
	       m_bPurgeDirecotorInNextLoop = false;
		   // 运行到此，说明程序的运行，已经没有逻辑代码需要处理了
	       purgeDirector();
	   }
	   else if (! m_bInvalid)
	    {
			// 屏幕绘制，并做一些相应的逻辑处理，其内部处理，这里暂且不做过多探讨
	        drawScene();
	    
	        // 这里实现了 cocos2d-x CCObject 对象的内存管理机制，对此有兴趣者，可以深入下去
	        CCPoolManager::sharedPoolManager()->pop();
	    }
	}

	// 弹出场景 CCScene
	void CCDirector::popScene(void)
	{
	    CCAssert(m_pRunningScene != NULL, "running scene should not null");
	
	    m_pobScenesStack->removeLastObject();
	    unsigned int c = m_pobScenesStack->count();
	
	    if (c == 0)
	    {
			// 如果没有场景，调用 end() 方法
	        end();
	    }
	    else
	    {
	        m_bSendCleanupToScene = true;
	        m_pNextScene = (CCScene*)m_pobScenesStack->objectAtIndex(c - 1);
	    }
	}

	void CCDirector::end()
	{
		// 在 end 方法中，设置了变量为 true，这所致的结果，在 mainLoop 函数中，达成了运行 purgeDirector 方法的条件
    	m_bPurgeDirecotorInNextLoop = true;
	}

	// 此方法做些收尾清理的工作
	void CCDirector::purgeDirector()
	{
		...
	    if (m_pRunningScene)
	    {
	        m_pRunningScene->onExit();
	        m_pRunningScene->cleanup();
	        m_pRunningScene->release();
	    }
		// 做一些清理的工作
	   ...
	    // OpenGL view

		// ###此句代码关键###
	    m_pobOpenGLView->end();
	    m_pobOpenGLView = NULL;
	
	    // delete CCDirector
	    release();
	}

	// 设置 openglview
	void CCDirector::setOpenGLView(CCEGLView *pobOpenGLView)
	{
	    CCAssert(pobOpenGLView, "opengl view should not be null");
	
	    if (m_pobOpenGLView != pobOpenGLView)
	    {
	        // EAGLView is not a CCObject
	        delete m_pobOpenGLView; // [openGLView_ release]
			// 为当前 CCDirector m_pobOpenGLView  赋值
	        m_pobOpenGLView = pobOpenGLView;
	
	        // set size
	        m_obWinSizeInPoints = m_pobOpenGLView->getDesignResolutionSize();
	        
	        createStatsLabel();
	        
	        if (m_pobOpenGLView)
	        {
	            setGLDefaultValues();
	        }  
	        
	        CHECK_GL_ERROR_DEBUG();
	
	        m_pobOpenGLView->setTouchDelegate(m_pTouchDispatcher);
	        m_pTouchDispatcher->setDispatchEvents(true);
	    }
	}
```	
	


游戏的运行以场景为基础，每时每刻都有一个场景正在运行，其内部有一个场景栈，遵循后进后出的原则，当我们显示的调用 end() 方法，或者弹出当前场景之时，其自动判断，如果没有场景存在，也会触发 end() 方法，以说明场景运行的结束，而游戏如果没有场景，就像演出没有了舞台，程序进入最后收尾的工作，通过修改变量 **m_bPurgeDirecotorInNextLoop** 促使在程序 **mainLoop** 方法之内调用 **purgeDirector** 方法。

### CCEGLView 的收尾工作

purgeDirector 方法之内，通过猜测与排查，最终定位到 **m_pobOpenGLView->end();** 方法，在这里结束了 cocos2d-x 游戏进程。而 m_pobOpenGLView 有时何时赋值，它的具体实现又在哪里呢？我们可以在 AppDelegate 的 **applicationDidFinishLaunching** 方法中找到如下代码：

``` c++
	// AppDelegate.cpp
	
	CCDirector *pDirector = CCDirector::sharedDirector();
	pDirector->setOpenGLView(CCEGLView::sharedOpenGLView());
```	

我们终于走到最后一步，看 CCEGLView 是如果负责收尾工作的:

``` c++
	// [cocos2dx-path]/cocos2dx/platform/linux.CCEGLView.cpp
	
	...
	CCEGLView* CCEGLView::sharedOpenGLView()
	{
	    static CCEGLView* s_pEglView = NULL;
	    if (s_pEglView == NULL)
	    {
	        s_pEglView = new CCEGLView();
	    }
	    return s_pEglView;
	}
	...

	// openglview 结束方法
	void CCEGLView::end()
	{
		/* Exits from GLFW */
		glfwTerminate();
		delete this;
		exit(0);
	}
```	

**end()** 方法很简单，只需要看到最后一句 **exit(0);** 就明白了。

### cocos2d-x 程序的结束流程

程序运行时期，由 **mainLoop** 方法维持运行着游戏之内的各个逻辑，当在弹出最后一个场景，或者直接调用 **CCDirector::end();** 方法后，触发游戏的清理工作，执行 **purgeDirector** 方法，从而结束了 CCEGLView（不同平台不同封装，PC使用OpenGl封装，移动终端封装的为 OpenGl ES） 的运行，调用其 **end()** 方法，从而直接执行 **exit(0);** 退出程序进程，从而结束了整个程序的运行。（Android 平台的 end() 方法内部通过Jni 方法 **terminateProcessJNI();** 调用 Java 实现的功能，其功能一样，直接结束了当前运行的进程）

从程序的 main 方法开始，再创建 AppDelegate 等对象，运行过程中确实通过 exit(0); 来退出程序。所以我们看到了 AppDelegate 构造函数被调用，而其析构函数没有被调用的现象。

**exit(0);** 的执行，意味着我们的程序完全结束，当然我们的进程资源也会被操作系统释放。但是注意，这里的 **在构造函数创建[资源]，并且在析构函数中释放[资源]** 并非绝对意义上的程序进程资源，在程序退出的时候，程序所使用的资源当然会被系统回收，但是如果我在构造函数调用网络接口初始化，析构在调用一次通知，所影响到的类似这种的 **非本地资源** 逻辑上的处理，而留下隐患。而通过理解 cocos2d-x 的运行机制，可以减少这种可能存在的隐患。

## cocos2d-x 的整体把握

在本文通过解决一个小疑问，而去分析 cocos2d-x 游戏的运行流程，当然其中很多细致末叶我们并没有深入下去。不去解决这个疑问也可以，知道没有调用析构函数，那我就不调用便是 （这也是简单的解决方法，也不用觉得这不可行 ）。这里只是借着这个疑问，对 cocos2d-x 的流程稍作探寻而已。也没有贴一堆 cocos2d-x 源码去分析，其思路也有迹可循。

什么是 cocos2d-x ,它是 cocos2d 一个 C++ 的实现，除 C++ 之外，有 python ，Objective-C 等其它语言的实现，那该怎么去理解 cocos2d ，可以这么理解，cocos2d 是一个编写 2D 游戏的通用形框架，这种框架提供了一个通用模型，而这种模型或者说架构是 **无关语言与平台** 的，说 cocos2d-x 使用 C++ 编写，其跨平台能力很强，但它能跑在浏览器上么？cocos2d 还是有着 html5  的实现，当然平台决定着语言的选择，而 cocos2d 能够适应这么多不同的语言和平台，其良好的设计，清晰的结构功不可没。 而对不同语言，对相同功能有着不同的封装，正如在本文问题中，在不同平台（Linux 和 Android），对相同功能有着不同的封装异曲同工。那么封装到最后，我们对 cocos2d  的理解就只剩下了，我们要写游戏，那么需要导演，场景、层、精灵、动作等 ~~ 组织好这个中之间的关系即可 ~
