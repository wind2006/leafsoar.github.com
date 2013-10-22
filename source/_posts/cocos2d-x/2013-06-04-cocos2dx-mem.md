---
layout: post
title: 深入理解 Cocos2d-x 内存管理
date: 2013-06-04 10:50
comments: true
categories: Cocos2d-x
---

如果 [Cocos2d-x 内存管理浅说](http://blog.leafsoar.com/archives/2013/05-22-23.html) 做为初步认识，而 [Cocos2d-x 内存管理的一种实现](http://blog.leafsoar.com/archives/2013/05-29-10.html)做为进阶使用，那么本文将详细的分析一下 Cocos2d-x 的内存管理的设计实现和原理。**知其然，知其所以然 ~**或者说：嗯，它这么做，一定是有原因的，体会设计者的用意，感同身受，如果是你，将会如何设计！~~

我觉得 **最好的学习方式是以自己的语言组织，说与别人听 ～** 这样对自己：更容易发现平时容易忽略的问题，对别人：或多或少也有所助益！以学习为目的，而别人的受益算是附带的效果，这样一个出发点 ~

由浅入深，总览全局（或者由整体到局部）是我喜欢的出发点，或者思考角度，我不喜欢拘泥于细节的实现，因为那会加大考虑问题的复杂度，所以 **把复杂的问题简单化，是必然的过程。** 那么本文就说说 Cocos2d-x 的架构是如何设计以方便内存管理的。从理论到实践 ~(当然是从我看问题的角度 :P，读者如有异议，欢迎讨论！文本使用 cocos2d-x 2.0.4 解说。)

<!-- more -->

-------------------------------------------------------------------------------

## 引用计数的由来

cocos2d-x 的世界是基于 **CCObject** 类构建的，其中的每个元素：层、场景、精灵等都是一个个 CCObject 的对象。所以 **内存管理的本质就是管理一个个 CCObject**。作为一个 cocos2d 的 C++ 移植版本，在它之前有很多其它语言的 **实现**，从架构层次来说，这与语言的实现无关（比如 CCNode 的节点树形关系，其它语言也可以实现，如果是内存方便，C# 等更是无需考虑），但就从内存管理方面来说，参考了 OC （Objective-C） 的内存管理实现。

一个简单的**自动管理原则**：**CCObject 内部维护着一个引用计数，引用计数为 0 就自动释放 ～**（如果么有直接做如 delete 之类的操作）。那么此时可以预见，**管理内存的实质就是管理这些 "引用计数" ** 了！使用 retain 和 release 方法对引用计数进行操作！

-------------------------------------------------------------------------------

## 为什么要有自动释放池 及其作用

我们知道 cocos2d-x 使用了自动释放池，自动管理对象，知其然！其所以然呢？**为什么需要自动释放池**，它在整个框架之中又起着什么样的作用！在了解这一点之前，我们需要 **知道 CCObject 从创建之初，到最终销毁**，经历了哪些过程。在此，一叶总结以下几点：

* 刚创建的对象，而 **为了保证在使用之前不会释放**（至少让它存活一帧），所以**自引用**（也就是初始为1）
* 为了确定是否 **实际使用**，所以需要在一个合适的时机，**解除自身引用。**
* 而这个何时的时机正是在**帧过度之时。**
* 帧过度之后的对象，**用则用矣，不用则弃！**
* 由于已经解除了自身引用，所以它的**引用被使用者管理**（一般而言，内部组成树形结构的链式反应，如 CCNode）。
* 链式反应，也就是，如果释放一个对象，也会释放它所引用的对象。

上面是一个对象的大致流程，我们将对象分为**两个时期**，一个是刚**创建时期**，自引用为 **1**（如果为 0 就会释放对象，这是基本原则，所以要大于 0） 的时期，另一个是**使用时期**。上面说到，为了保证创建时期的对象不被销毁，所以自引用(**并没有实际的使用**)初始化为 1，这就意味着我们需要一个合适的时机，来解除这样的自引用。

**何时？**在帧过度之时！(这样可保证当前帧能正确使用对象而没有被销毁。)**怎么样释放？**由于是自引用，我们并不能通过其它方式访问到它，所以就有了自动释放池，我们 **变相的将“自引用”转化“自动释放池引用”，来标记一个 “创建时期的对象”**。然后在帧过度之时，通过自动释放池管理，统一释放 “释放池引用”，也就意味着，去除了“自身引用”。**帧过度之后的对象，才是真正的被使用者所管理。** 下面我们用代码来解释上述过程。

通常我们使用 `create();` 方法来创建一个自动管理的对象，而其内部实际操作如下：

``` c++
	// 初始化一个对象
	static CCObject* create() 
	{
		// new CCObject 对象
		CCObject *pRet = new CCObject(); 
	    if (pRet && pRet->init()) 
	    {
			// 添加到自动释放池
	        pRet->autorelease(); 
	        return pRet; 
	    } 
	    else 
	    { 
	        delete pRet; 
	        pRet = 0; 
	        return 0; 
	    } 
	}

	// 我们看到初始化的对象 自引用 m_uReference = 1
	CCObject::CCObject(void)
	:m_uAutoReleaseCount(0)
	,m_uReference(1) // when the object is created, the reference count of it is 1
	,m_nLuaID(0)
	{
	    static unsigned int uObjectCount = 0;
	
	    m_uID = ++uObjectCount;
	}

	// 标记为自动释放对象
	CCObject* CCObject::autorelease(void)
	{
		// 添加到自动释放池
	    CCPoolManager::sharedPoolManager()->addObject(this);
	    return this;
	}

	// 继续跟踪
	void CCPoolManager::addObject(CCObject* pObject)
	{
	    getCurReleasePool()->addObject(pObject);
	}

	// 添加到自动释放池的实际操作
	void CCAutoreleasePool::addObject(CCObject* pObject)
	{
		// 内部是由一个 CCArray 维护自动释放对象，并且此操作 会使引用 + 1
	    m_pManagedObjectArray->addObject(pObject);

		// 由于初始化 引用为 1，上面又有操作，所以引用至少为 2 （可能还被其它所引用）
	    CCAssert(pObject->m_uReference > 1, "reference count should be greater than 1");
	    ++(pObject->m_uAutoReleaseCount);
		// 变相的将自身引用转化为释放池引用，所以减 1
	    pObject->release(); // no ref count, in this case autorelease pool added.
	}
```	

上面便是通过 `create() ` 方法创建对象的过程。文中说到，一个合适的时机，解除自身引用（也就是释放池引用），那这又是在何时进行的呢？程序的运行有一个主循环，控制着每一帧的操作，在每一帧画面画完之时会自动调用 `CCPoolManager::sharedPoolManager()->pop();` 方法 ( 具体可参见文章[Cocos2d-x 程序是如何开始运行与结束的](http://blog.leafsoar.com/archives/2013/05-05-23.html) ，这里我们只要知道每一帧结束都会调用 pop() 方法)，来自动清理 **创建时期** 的引用。现在我们就来看看 `pop()` 的方法实现：

``` c++	
	void CCPoolManager::pop()
	{
	    if (! m_pCurReleasePool)
	    {
	        return;
	    }

		// 当前释放池个数，pop 使用栈结构
	     int nCount = m_pReleasePoolStack->count();
	 	// 释放池当中存放的都是 创建时期 对象，此时解除释放池引用
	    m_pCurReleasePool->clear();

		// 当前释放池，出栈，在这里可以看到判断 nCount 是否大于 1，文后将会对此做具体说明
	      if(nCount > 1)
	      {
	        m_pReleasePoolStack->removeObjectAtIndex(nCount-1);
	
	//         if(nCount > 1)
	//         {
	//             m_pCurReleasePool = m_pReleasePoolStack->objectAtIndex(nCount - 2);
	//             return;
	//         }
	        m_pCurReleasePool = (CCAutoreleasePool*)m_pReleasePoolStack->objectAtIndex(nCount - 2);
	    }
	
	    /*m_pCurReleasePool = NULL;*/
	}

	// 释放池引用清理工作
	void CCAutoreleasePool::clear()
	{
		// 如果释放池存在 创建时期 的对象
	    if(m_pManagedObjectArray->count() > 0)
	    {
	        //CCAutoreleasePool* pReleasePool;
	#ifdef _DEBUG
	        int nIndex = m_pManagedObjectArray->count() - 1;
	#endif
	
	        CCObject* pObj = NULL;
	        CCARRAY_FOREACH_REVERSE(m_pManagedObjectArray, pObj)
	        {
	            if(!pObj)
	                break;
	
	            --(pObj->m_uAutoReleaseCount);
	            //(*it)->release();
	            //delete (*it);
	#ifdef _DEBUG
	            nIndex--;
	#endif
	        }
			// 移除释放池对创建时期对象的引用，从而使对象交由使用者全权管理
	        m_pManagedObjectArray->removeAllObjects();
	    }
	}
```	

**到这里，自动释放池的作用也就完成了！** 可以说创建的对象在一帧 (**但有特殊情况，下一段说明**) 之后就完全脱离了 **自动释放池的控制**，自动释放池，对对象的管理也就在 **创建时期起着作用**！之后便交由使用者管理，释放。

-------------------------------------------------------------------------------

## 对"释放池"的管理说明

我们知道了释放池管理着 **创建时期** 的对象，那么对于释放池本身是如何管理的？我们知道对于释放池，只需要有一个就已经能够满足我们的需求了，而在 cocos2d-x 的设计中，使用了集合管理 **一堆** 释放池。而在实际，它们又发挥了多大的用处？

``` c++
	// 释放池管理接口
	class CC_DLL CCPoolManager
	{
		// 释放池对象集合
	    CCArray*    m_pReleasePoolStack;
		// 当前操作释放池
	    CCAutoreleasePool*                    m_pCurReleasePool;

		// 获取当前释放池
	    CCAutoreleasePool* getCurReleasePool();
	public:
	    CCPoolManager();
	    ~CCPoolManager();
	    void finalize();
	    void push();
	    void pop();
	
	    void removeObject(CCObject* pObject);
		// 添加一个 创建时期 对象
	    void addObject(CCObject* pObject);
	
	    static CCPoolManager* sharedPoolManager();
	    static void purgePoolManager();
	
	    friend class CCAutoreleasePool;
	};

	// 我们从 addObject 开始看起，由上文可以 addObject 是由 CCObject 的 autorelease 自动调用的
	void CCPoolManager::addObject(CCObject* pObject)
	{
	    getCurReleasePool()->addObject(pObject);
	}

	CCAutoreleasePool* CCPoolManager::getCurReleasePool()
	{
		// 如果当前释放池为空
	    if(!m_pCurReleasePool)
	    {
			// 添加一个
	        push();
	    }
	
	    CCAssert(m_pCurReleasePool, "current auto release pool should not be null");
	
	    return m_pCurReleasePool;
	}

	void CCPoolManager::push()
	{
	    CCAutoreleasePool* pPool = new CCAutoreleasePool();       //ref = 1
	    m_pCurReleasePool = pPool;
		// 像集合添加一个新的释放池
	    m_pReleasePoolStack->addObject(pPool);                   //ref = 2
	
	    pPool->release();                                       //ref = 1
	}
```	

从 addObject 开始分析，我们知道在 addObject 之前，会首先判断是否有当前的释放池，如果没有则创建，如果有，则直接使用，可想而知，在任何使用，任何情况，通过 addObject 只需要创建一个释放池便已经足够使用了。事实上也是如此。再来看 pop 方法。

``` c++
	void CCPoolManager::pop()
	{
	    if (! m_pCurReleasePool)
	    {
	        return;
	    }
	
	     int nCount = m_pReleasePoolStack->count();
	 	// 清楚对 创建对象 的引用
	    m_pCurReleasePool->clear();

		// 如果大于 1，这也保证着，在任何时候，总有一个释放池是可以使用的
	      if(nCount > 1)
	      {
			  // 移除当前的释放池
	        m_pReleasePoolStack->removeObjectAtIndex(nCount-1);
	
	//         if(nCount > 1)
	//         {
	//             m_pCurReleasePool = m_pReleasePoolStack->objectAtIndex(nCount - 2);
	//             return;
	//         }
			// 将当前释放池设定为前一个释放池，也就是 “出栈”的操作
	        m_pCurReleasePool = (CCAutoreleasePool*)m_pReleasePoolStack->objectAtIndex(nCount - 2);
	    }
	
	    /*m_pCurReleasePool = NULL;*/
	}
```	

**看到这里** 我就不解了！什么情况下才能用到多个释放池？按照设计的逻辑根本用不到。带着这个疑问，我在 `CCPoolManager::push()` 方法之内添加了一句话打印（修改源代码） `CCLog("这里要长长长的 **********");` ，然后重新编译源文件，运行程序，发现实际的使用中，push 只被调用了两次！我们知道，通过 addObject 可能会自动调用 `push()` 一次，但也仅有一次，所以一定是哪里手动调用了 `push()` 方法，才会出现这种情况，所以我继续翻看源代码，定位到了 `bool CCDirector::init(void)` 方法，在这里进行了游戏的全局初始化相关工作：

``` c++
	bool CCDirector::init(void)
	{
		CCLOG("cocos2d: %s", cocos2dVersion());
	
		...
		...
		m_dOldAnimationInterval = m_dAnimationInterval = 1.0 / kDefaultFPS;    
	    m_pobScenesStack = new CCArray();
	    m_pobScenesStack->init();
	
		...
		...
	    m_fContentScaleFactor = 1.0f;
	
		...
		...
	    // touchDispatcher
	    m_pTouchDispatcher = new CCTouchDispatcher();
	    m_pTouchDispatcher->init();
	
	    // KeypadDispatcher
	    m_pKeypadDispatcher = new CCKeypadDispatcher();
	
	    // Accelerometer
	    m_pAccelerometer = new CCAccelerometer();
	
	
	    // 这里手动调用了 push 方法，而在这之前的初始化过程中，间接的使用了 CCObject 的 autorelease，已经触发过一次 push 方法
	    CCPoolManager::sharedPoolManager()->push();
	
	    return true;
	}
```	

**所以我们便能够看到 push 方法被调用了两次**，但其实如果我们把这里的手动调用放在方法的开始处，或者干脆就不使用 `CCPoolManager::sharedPoolManager()->push();` ，对程序也没任何影响，这样从头到尾，**只创建了一个自动释放池，而这里多创建的一个并没有多大的用处。** 或者用处不甚明显，因为多创建一个释放池是有其效果的，效果具体体现在哪里，那就是 **可以使调用 push() 方法之前的对象，多存活一帧。**，因为 pop 方法只对当前释放池做了 clear 释放。为了方便起见，我们使用 [Cocos2d-x 内存管理浅说](http://blog.leafsoar.com/archives/2013/05-22-23.html) 里面的方法观察每一帧的情况，看下面测试代码：

``` c++
	// 关键代码如下
	CCLog("update index: %d", updateCount);
	
	// 在不同的帧做相关操作，以便观察
	if (updateCount == 1) {
		// 创建一个自动管理对象
		layer = LSLayer::create();
		// 创建一个新的自动释放池
		CCPoolManager::sharedPoolManager()->push();
		// 再创建一个自动管理对象
		sprite = LSSprite::create();
	} else if (updateCount == 2) {
	
	} else if (updateCount == 3) {
	
	}
	
	CCLog("update index: %d end", updateCount);

	/// 打印代码如下
	cocos2d-x debug info [update index: 1]
	// 第一帧创建了两个自动管理对象
	cocos2d-x debug info [LSLayer().()]
	cocos2d-x debug info [LSSprite().()]
	cocos2d-x debug info [update index: 1 end]
	// 第一个过度帧只释放了 sprite 对象
	cocos2d-x debug info [LSSprite().~()]
	cocos2d-x debug info [update index: 2]
	cocos2d-x debug info [update index: 2 end]
	// 第二个过度帧释放了 layer 对象
	cocos2d-x debug info [LSLayer().~()]
	cocos2d-x debug info [update index: 3]
	cocos2d-x debug info [update index: 3 end]
```	

可以对比 sprite 和 layer 对象，两个对象被放在了不同的自动释放池之中。这就是 手动调用 `push()` 方法所能达到的效果，至于怎么利用这个特性，**帮助我们完成特殊的功能？我想还是不用了**，这会增加我们程序设计的 **复杂度**，在我看来，甚至想把，cocos2d-x 2.0.4 中那唯一一次调用的 `push()` 给删了，以保持简单（程序的第一次初始化“可能”会用到这个特性，不过目测是没有多大关系的了 : P），在这里只系统通过这个例子理解 自动释放池是怎样被管理的即可！

从自动释放池管理 **创建时期** 对象，再到对释放池的管理，我们已经大概了解了一个对象的生命周期经历了哪些！ 下面简单说说 **使用时期** 的对象管理。


-------------------------------------------------------------------------------

## 树形结构的链式反应

文中我们知道了，自动释放池的存在意义，在于对象 **创建时期** 的处理，而仅仅理解了自动释放池，对于我们使用 cocos2d-x 不够，远远不够！自动释放池只是解决对象初始化的问题，仅此而已，而要在整个使用过程中，相对的自动化管理，那么必须理解两个概念，**树形结构** 和 **链式反应** （链式反应，不错的说法，就像原子弹爆炸一样，一传十，十传百 ：P）

我们当前运行这一个场景，场景初始化，添加了很多层，层里面有其它的层或者精灵，而这些都是 CCNode 节点，以场景为根，形成一个树形结构，场景初始化之后（一帧之后），这些节点将完全 **依附** (内部通过 retain) 在这个树形结构之上，全权交由树来管理，当我们 **砍去一个树枝**，或者将树 **连根拔起**，那么在它之上的“子节点”也会跟着去除(内部通过 release)，这便是链式反应。

[Cocos2d-x 内存管理的一种实现](http://blog.leafsoar.com/archives/2013/05-29-10.html)，此文这种实现的本质既是 **强化**这种 **链式反应**，也是解决内存可能出错的一个解决方案。如下（前文片段，具体详见前文）：

``` c++
	// 方式一：那么我们的使用过程
	LUser* lu = LUser::create();
	lu->m_sSprite = CCSprite::create("a.png");
	// 如果这里不 retain  则以后就用不到了
	lu->m_sSprite->retain();
	
	// 方式二：使用方法
	LUser* lu = LUser::create();
	lu->m_sUserName = "一叶";
	// 这里的 sprite 会随着 lu 的消亡而消亡，不用管释放问题了
	lu->setSprite(CCSprite::create("a.png"));
```	

我们看到方式二相比方式一的设计，它通过 setSprite 内部对 sprite 本身 retain，从而实现**链式反应**，而不是直接使用 `lu->m_sSprite->retain();`，这样的好处是，我只要想着释放 LUser，而不用考虑LUser 内部 sprite 的引用情况就行了。如此才能把 cocos2d-x 内存的自动管理特性完全发挥 ~

而要实现这样管理的一个明显特征就是，隐藏 `retain` 和 `release` 操作 ~

-------------------------------------------------------------------------------

## 稍作总结

关于 cocos2d-x 的内存管理从使用到原理，系列文章就到这里了！（三篇也算系列 = =!） 由表象到内部的思考探索过程，其实在 **浅说** 当中对 cocos2d-x 的使用，便已经能够知晓内部细节设计之一二，透过现象看本质！三篇文章包含了，使用浅说（简单的测试），一种防止内存泄漏的设计（加强链式反应），最后纵览 cocos2d-x 的内存管理框架，对 CCObject 的生命周期做了简单的说明，当然其中还是隐藏一些细节的，比如管理都是用 CCArray 来管理，但我们并没有对 CCArray 做介绍，它是如何添加元素，如何引用等。在任何时候我们只针对一个问题进行思考，那我们该把 CCArray 这样的辅助工具类放在何处，如果你了解当然最好，不过不了解，那便 **存疑** ，然后对相应的问题，分而治之 ~

**存疑** 可以帮助一叶在某个时刻只针对某一个问题进行思考，从而使问题变的简单。对文中所涉及的到的两个类 `CCPoolManager` 和 `CCAutoreleasePool` 其中所有的方法并没有面面俱到，当然有了整体思路，去 **填充那些** 小疑问将会变得简单。
