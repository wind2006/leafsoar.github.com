---
layout: post
title: 类 Android 多级日志系统应用
date: 2013-05-15 17:30
comments: true
categories: Cocos2d-x
---

在 cocos2d-x 中实现类 Android 多级日志系统！写程序，免不了调试打印 Log ，而一个方便的日志系统，可以提高不少开发的效率，写过 Android 应用的朋友应该了解在 Android 中的日志包含很多等级，查看日志的时候可以指定日志的级别，从而过滤一些无用的信息，能够更为快速的定位问题的所在。而在 cocos2d-x 的开发中，统一使用 **CCLog**，只能算是最基本的功能，既然有好的思想可以借鉴，何乐而不为呢～

先看看最终实现的效果，我们使用简单封装过的方法调用打印日志系统（Leafsoar Log:LSLog）：

<!-- more -->

``` bash
    LSLog::verbose("博客名称： %s","无间落叶");
    LSLog::debug("博客地址： %s", "http://blog.leafsoar.com");
    LSLog::info("基本信息： %s", "吾名 一叶");
    LSLog::warn("多出警告： %s %s", "警告一", "警告二");
    LSLog::error("坐标错误： (%f, %f)", 800.0f, 600.0f);

	// Linux 系统下打印信息 （非 Android）
	cocos2d-x debug info [(verbose)	 :博客名称： 无间落叶]
	cocos2d-x debug info [(debug)		 :博客地址： http://blog.leafsoar.com]
	cocos2d-x debug info [(info)		 :基本信息： 吾名 一叶]
	cocos2d-x debug info [(warn)		 :多出警告： 警告一 警告二]
	cocos2d-x debug info [(error)		 :坐标错误： (800.000000, 600.000000)]
```

在 Android 的 Logcat 下显示效果：
![图片](/images/2013/cocos2d-x-log.png)

在 Android 平台，可以打印所有信息，并且通过 LogCat 过滤信息，还有相应的高亮显示。而这些都已经在 **LSLog** 内部通过 Jni 调用实现，如需如上所示的调用即可。

通过日志的等级划分过滤，可以让我们在调试某一个功能时只关注某一类调试信息，比如只显示警告信息和错误信息，而不是哗啦啦所有的日志都打印出来，然后一行一行读，查看，排错。

而在非 Android 平台，并没有如 LogCat 这样的工具，其它平台我不知晓，但我所在的 Linux 平台显然之用默认的日志打印，所以在 LSLog 的内部实现之中，**通过一个开关控制显示哪些级别的日志信息，这个级别可以任意的定义。** 看下面实现[源码获取](https://github.com/leafsoar/ls-cocos2d-x/tree/master/Learn/Classes)：

``` c++
	// LSLog.h

	#ifndef LSLOG_H_
	#define LSLOG_H_
	
	#include "cocos2d.h"

	// 日志级别，也可根据自己需要修改添加类别
	enum{
		LSLOG_VERBOSE = 0,
		LSLOG_DEBUG,
		LSLOG_INFO,
		LSLOG_WARN,
		LSLOG_ERROR,
		LSLOG_COUNT,
	};

	// 打印日志类别前缀
	const std::string lsLog_name[LSLOG_COUNT] = {
			"(verbose)\t",
			"(debug)\t\t",
			"(info)\t\t",
			"(warn)\t\t",
			"(error)\t\t"
	};

	// 不同级别对应的 Android Jni 实现方法名称
	const std::string lsLog_androidMethod[LSLOG_COUNT] = {
			"v",
			"d",
			"i",
			"w",
			"e"
	};
	
	/**
	 @brief 自定义日志系统，前期使用，以后可以扩展优化
	 */
	class LSLog: public cocos2d::CCObject {
	public:
		/// verbose 详细日志，一般常用的打印信息
		static void verbose(const char * pszFormat, ...);
		/// debug 调试 ,调试过程所注意的信息
		static void debug(const char * pszFormat, ...);
		/// info 一般信息,
		static void info(const char * pszFormat, ...);
		///  warn 警告信息
		static void warn(const char * pszFormat, ...);
		/// error 错误信息
		static void error(const char * pszFormat, ...);
	private:
		// 需要显示的日志级别定义
		static const int LOG_VALUE;
		// 打印日志方法
		static void printLog(int type, const char* format, va_list ap);
		// Android 平台日志打印
		static void printAndroidLog(const char* methodName, const char* log);
	};
	
	#endif /* LSLOG_H_ */
	
	// LSLog.cpp

	#include "LSLog.h"
	
	#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
	
	#include "platform/android/jni/JniHelper.h"
	#include "platform/android/jni/Java_org_cocos2dx_lib_Cocos2dxHelper.h"
	#include <jni.h>
	
	#endif
	
	USING_NS_CC;
	
	#define kMaxStringLen (1024*100)
	#define LOG_V 1
	#define LOG_D 2
	#define LOG_I 4
	#define LOG_W 8
	#define LOG_E 16
	
	/// 需要打印的日志级别，根据 LOG_VALUE 的设置打印不同级别的日志
	//const int LSLog::LOG_VALUE = LOG_V | LOG_D | LOG_I | LOG_W | LOG_E;
	//const int LSLog::LOG_VALUE = LOG_D | LOG_I | LOG_W | LOG_E;
	const int LSLog::LOG_VALUE = LOG_I | LOG_W | LOG_E;
	//const int LSLog::LOG_VALUE = LOG_W | LOG_E;
	//const int LSLog::LOG_VALUE = LOG_E;
	//const int LSLog::LOG_VALUE = 0;

	// 这里灵活控制，可以只打印某一个级别
	//const int LSLog::LOG_VALUE = LOG_D;
	
	void LSLog::verbose(const char * pszFormat, ...) {
		if (LOG_V & LOG_VALUE) {
			va_list ap;
			va_start(ap, pszFormat);
			LSLog::printLog(LSLOG_VERBOSE, pszFormat, ap);
			va_end(ap);
		}
	}
	
	void LSLog::debug(const char* pszFormat, ...) {
		if (LOG_D & LOG_VALUE) {
			va_list ap;
			va_start(ap, pszFormat);
			LSLog::printLog(LSLOG_DEBUG, pszFormat, ap);
			va_end(ap);
		}
	}
	
	void LSLog::info(const char* pszFormat, ...) {
		if (LOG_I & LOG_VALUE) {
			va_list ap;
			va_start(ap, pszFormat);
			LSLog::printLog(LSLOG_INFO, pszFormat, ap);
			va_end(ap);
		}
	}
	
	void LSLog::warn(const char* pszFormat, ...) {
		if (LOG_W & LOG_VALUE) {
			va_list ap;
			va_start(ap, pszFormat);
			LSLog::printLog(LSLOG_WARN, pszFormat, ap);
			va_end(ap);
		}
	}
	
	void LSLog::error(const char* pszFormat, ...) {
		if (LOG_E & LOG_VALUE) {
			va_list ap;
			va_start(ap, pszFormat);
			LSLog::printLog(LSLOG_ERROR, pszFormat, ap);
			va_end(ap);
		}
	}
	
	void LSLog::printLog(int type, const char* format, va_list ap) {
		char* pBuf = (char*) malloc(kMaxStringLen);
		std::string mstr;
		if (pBuf != NULL) {
			vsnprintf(pBuf, kMaxStringLen, format, ap);
			mstr = pBuf;
			free(pBuf);
		}
	#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
		printAndroidLog(lsLog_androidMethod[type].c_str(), mstr.c_str());
	#else
		CCLog("%s :%s", lsLog_name[type].c_str(), mstr.c_str());
	#endif
	}
	
	void LSLog::printAndroidLog(const char* methodName, const char* log) {
	#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
		JniMethodInfo t;
		bool isHave = JniHelper::getStaticMethodInfo(t,
				"android/util/Log",
				methodName,
				"(Ljava/lang/String;Ljava/lang/String;)I");
		if (isHave)
		{
			jstring jTitle = t.env->NewStringUTF("cocos2d-x");
			jstring jMsg = t.env->NewStringUTF(
					log);
			t.env->CallStaticVoidMethod(t.classID, t.methodID, jTitle,
					jMsg);
			t.env->DeleteLocalRef(jTitle);
			t.env->DeleteLocalRef(jMsg);
		}
		else
		{
			CCLog("the jni method is not exits");
		}
	#endif
	}
```	

这使前期开发方便了许多，而使用自定义的日志系统，另一个好处就是后期扩展，比如我们想将日志保存文本，收集错误信息等，都可以只通过修改日志内部方法的实现即可完成 ~
