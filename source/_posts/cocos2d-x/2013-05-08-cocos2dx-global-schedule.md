---
layout: post
title: 实现 Cocos2d-x 全局定时器
date: 2013-05-8 21:30
comments: true
categories: Cocos2d-x
---

cocos2d-x 中有自己的定时器实现，一般用法是在场景，层等内部实现，定时器的生命周期随着它们的消亡而消亡，就运行周期而言，相对最长的是场景，如果在多个场景切换并且保持定时器的运行，那我们就需要定义一个自己的  **全局定时器**。

平时所使用的定时器，我们可以直接使用，是因为 **CCNode** 帮我们实现了定时器的封装，一个简单的做法，是定义个全局的静态 CCNode 对象，在程序运行之初初始化，并执行其定时器任务，而不由任何场景所管理即可实现，但在这里，一叶对定时器的内部实现稍作了解后，封装了自己实现的全局定时解决方案，代码如下([获取源码](https://github.com/leafsoar/ls-cocos2d-x/tree/master/Learn/Classes/GlobalSchedule) )：

<!-- more -->

``` c++
	////////////////////
	// GlobalSchedule.h
	////////////////////	
	
	#ifndef GLOBALSCHEDULE_H_
	#define GLOBALSCHEDULE_H_
	
	#include "cocos2d.h"
	
	USING_NS_CC;
	
	/**
	 *	全局定时器
	 */
	class GlobalSchedule: public CCObject {
	public:
		// 开始全局定时器 fInterval: 时间间隔 ; fDelay: 延迟运行
		static void start(float fInterval = 0.0f, float fDelay = 0.0f);
		// 停止全局定时器
		static void stop();
		// 全局定时器暂停
		static void pause();
		// 全局定时器暂停恢复
		static void resume();
	
		// 全局定时器主逻辑实现
		void globalUpdate();
	
	private:
		// 构造函数私有化，只能通过 start 来启用全局定时器
		GlobalSchedule(float fInterval, float fDelay);
		~GlobalSchedule();
	
		// 静态变量保持单例
		static GlobalSchedule* m_pSchedule;
	};
	
	#endif /* GLOBALSCHEDULE_H_ */
	
	/////////////////////
	// GlobalSchedule.cpp
	/////////////////////	
	
	#include "GlobalSchedule.h"
	
	#define SCHEDULE CCDirector::sharedDirector()->getScheduler()
	
	GlobalSchedule* GlobalSchedule::m_pSchedule = NULL;
	
	GlobalSchedule::GlobalSchedule(float fInterval, float fDelay) {
		CCLog("GlobalSchedule()");
	
		CCAssert(!m_pSchedule, "已定义，不能重复定义");
	
		SCHEDULE->scheduleSelector(
				schedule_selector(GlobalSchedule::globalUpdate), this, fInterval,
				false,
				kCCRepeatForever, fDelay);
	
		m_pSchedule = this;
	}
	
	GlobalSchedule::~GlobalSchedule() {
		CCLog("GlobalSchedule().~()");
	
		SCHEDULE->unscheduleSelector(
				schedule_selector(GlobalSchedule::globalUpdate), this);
	}
	
	void GlobalSchedule::globalUpdate() {
		// 这里写全局定时器的逻辑处理代码
		CCLog("global update");
	}
	
	void GlobalSchedule::start(float fInterval, float fDelay) {
		new GlobalSchedule(fInterval, fDelay);
	}
	
	void GlobalSchedule::stop() {
		CCLog("GlobalSchedule().stop()");
	
		CCAssert(m_pSchedule, "未定义");
		CC_SAFE_DELETE(m_pSchedule);
	}
	
	void GlobalSchedule::pause() {
		CCLog("GlobalSchedule().pause()");
	
		CCAssert(m_pSchedule, "未定义");
		SCHEDULE->pauseTarget(m_pSchedule);
	}
	
	void GlobalSchedule::resume() {
		CCLog("GlobalSchedule().resume()");
	
		CCAssert(m_pSchedule, " 未定义");
		SCHEDULE->resumeTarget(m_pSchedule);
	}
```

<font color="red" >注意事项：根据一朋友的使用反馈（多谢这位朋友:P），以上代码并不能在 2.1.x 版本如期运行，原因为 scheduleSelector（） 中的形参位置有变，请根据实际情况修改此构造函数内调用定时器 实参 的位置！</font> **除非说明，本博客默认使用当前稳定版：2.0.4**
### 使用方法
这样的封装，在使用的时候只要填写 **globalUpdate()** 方法，处理具体的逻辑，然后在 **AppDelegate** 的 **applicationDidFinishLaunching** 调用如下代码:

``` c++
	// 启动定时器
	GlobalSchedule::start();
	// 启动定时器，每 0.2 秒间隔执行
	GlobalSchedule::start(0.2f);
	// 每 0.5 秒间隔运行，延迟 3 秒启动
	GlobalSchedule::start(0.5f, 3.0f);
```	

注意，**start()** 不论调用哪个重载的方法， 只能调用一次。当然可以在调用 **stop()** 方法重新调用 **start()** 启动定时器，方法的重载实现了定时器的时间间隔和延迟运行，并实现了定时器的暂停和恢复功能。

### 什么时候结束

推荐的结束时机为在最后一个场景结束之时：

``` c++
	CCDirector::sharedDirector()->end();
	GlobalSchedule::stop();
```	

在前面的博文 [Cocos2d-x 程序是如何开始运行与结束的](http://blog.leafsoar.com/archives/2013/05-05-23.html)，我们分析了 cocos2d-x 程序运行的始末，故推荐在此时调用停止定时器的方法。

有了全局定时器我们能做什么？这就要问你为何而实现全局定时器了 : P
