---
layout: post
title: 多层 UI 触摸事件的轻量级设计
date: 2013-05-25 10:10
comments: true
categories: Cocos2d-x
---

**轻量级**:一叶非常喜欢的名词，在重量级和轻量级之间，如果做选择的话，一定会选择轻量级，它的特点首先是设计简单小巧，使用方便，更具有灵活性，扩展方便。重量级则大而丰富，全面，但略显笨重，在程序设计之初大多需要全盘考虑。而轻重之间的概念是相对而言，并没有严格的界限。

## Cocos2d-x 触摸事件机制概论

在 cocos2d-x 使用触摸来触发一些操作是很常用的功能，如果界面非常简单，只需要启用相应层的触摸功能，并处理其触摸事件即可，而如果界面的 UI 复杂，多层管理，又有着隐藏控制，灵活多变，比如 MMO 游戏，当然手游不会 **那么** 复杂，那么现有的机制实现起来就显得捉襟见肘了，即便实现，也很难维护，而一个简单的方式是 **只在场景的 基层 接受触摸消息，然后由此基层向上层发送触摸的消息**，上层再根据实际情况进行处理，判断可触摸元素优先级，是否隐藏，返回处理结果，再一层层向下传递，保证实际的操作是我们所期望的。

<!-- more -->

在基层接受触摸消息，然后向上层发送触摸消息，而在 cocos2d-x 中并没有这样一个机制，所以已经有人基于 cocos2d-x 实现了这样一个机制，比如我们 **实现自己的场景、层等，和自己的 一套层级控制**，这个控制具有传递触摸消息的机制，但是这样我们就不能继续使用原有的层级管理机制。还有 **通过修改 cocos2d-x 的源代码，达到这样的效果** ，而这样的 **侵入 API** 的方式不甚可取，无论如何，这样的方式略显笨重，使用之前需要做很多工作，算是重量级的设计思路吧 ~

为了使用的简单，并基于以上考虑，所以想到要设计一个 **轻量级 的复杂 UI 触摸事件管理机制**。首先从使用者角度考虑，要使用简单，嵌入到现有 cocos2d-x 方便，并没有什么复杂的特性，其次从设计角度考虑，充分利用 cocos2d-x 现有的特性，保持自身的简洁，关于此点，将会在后面的文章内容体验。

## 抽象：轻量级设计的可行性分析

触摸事件，从触摸开始，到有效点击，然后触发点击事件，从这么一个过程我们提取 **两个抽象概念**，而这两个概念将是我们的设计核心内容。首先要有 **“可触摸对象”** 类，也就是界面上一个可点击操作的元素，我们知道在 cocos2d-x 中有 CCScene、CCLayer、CCNode 等，大多情况都只是作为 **容器** 使用，本身并不处理触摸操作，而这些内容我们完全不用关注。还需要一个**“可触摸对象事件管理对象”** 类型，就简称 **管理类** 吧，管理类管理可触摸对象。

现在我们设想这样一种情况，场景基层作为管理层，在这之中维护着一个 “可触摸对象”的集合，当我们创建一个可触摸对象的并把它添加到界面上之时，我们将它添加到这个集合中，当然这个可触摸对象包含一些属性标示，比如设定事件 Id 等。无论界面怎么布局，层次关系如何复杂，我们只需要关注这个可触摸对象的集合即可。好了，现在我们点击界面，通过场景基层接受触摸消息，获得点击的点，**现在我们要做的就是判断哪个可触摸对象是有效点击就行了**，从集合中找出有效点击的对象是很容易的。我们可以做一些判断，以确定哪个元素是有效点击，从而触发它的事件，而这个触发操作统一由管理层触发，现在我们来定一些有效点击的规则，并且这个规则是可以根据自己需要添加修改的：

* 可触摸对象有个可触摸的范围（ContentSize），判断触摸的点是否在可触摸范围之内
* 可触摸对象是否正在运行（IsRunning），排除了，已经从界面移除可触摸对象，可能没有及时释放而触发的情形
* 可触摸对象是否隐藏 （IsVisible），如果不可见，当然无效点击
* 可触摸对象的父层是否有隐藏，只需要不停的获取父层，判断是否存在以藏即可
* 其它判断，自己添加定义 ~~~

从集合中找出满足以上条件的元素是可行的，如果满足条件的有多个元素呢？这是可能的，比如两个可触摸对象的可触摸范围重叠，这是我们就需要对这两个元素做优先级比较了，如何比较？我们知道任意两个可触摸对象，是被添加的场景基层中的 **树形结构**，我们只需要分析这个树形结构，找到这两个可触摸节点的优先级即可，过程简说：找出两个节点最近的共同父节点，从而定位到此父节点下，两个元素所在的子节点，此两节点首先根据 ZOrder 判断优先，如果 ZOrder 相同，判断节点在父节点的索引位置，从而判断优先级。

至此我们就能从可触摸对象集合中找到 **一个** 最终满足所有条件的对象，有了这个对象，我们就可以够精确的触发其触摸事件！

## 一个简单的设计雏形

雏形的设计一切从简，200 行代码左右。首先定义了一个 **可触摸对象** 类型 **LsTouch** ，它标示一个可触摸的对象，其中有一个 CCSprite 属性，显示和判断可点击范围都靠它（简单起见，这里可以定义自己的属性扩充，满足各种需要），还包含一个事件 Id 属性，知道触发什么事件（可以添加如事件类型属性等方便事件的处理）。另外定义了 **LsTouchEvent** 事件处理类，也是管理类，在使用的时候，场景基层实现它，并实现 **touchEventAction** 方法，此方法用户处理事件响应，而在 **ccTouchesBegan** 方法之内调用 LsToucheEvent 定义的 `sendTouchEvent(CCTouch* ccTouch)` 方法，传递 **CCTouch** 参数，之后方法内部会自动判断有效点击，并自动触发 touchEventAction 方法。

在介绍实现之前，先通过简单的代码看看使用方法，从使用过程中体现它的简洁：

``` c++
	// 场景基层定义，实现 LsTouchEvent 的 touchEventAction 事件响应方法即可
	class TouchEventTest: public CCLayer , public LsTouchEvent{
	public:
		CREATE_FUNC(TouchEventTest)
		;
		virtual bool init();
	
		virtual void ccTouchesBegan(CCSet *pTouches, CCEvent *pEvent);
	
		virtual void touchEventAction(LsTouch* touch);
	};

	// TouchEventTest 实现
	bool TouchEventTest::init() {
		bool bRef = false;
		do {
			CC_BREAK_IF(!CCLayer::init());
	
			// 启用触摸
			setTouchEnabled(true);
	
			CCSize winSize = CCDirector::sharedDirector()->getWinSize();
			CCPoint center = ccp(winSize.width/ 2, winSize.height / 2);
	
			// 创建可触摸精灵
			LsTouch* lt = LsTouch::create();
			// 设置位置
			lt->setPosition(center);
			// 设置显示精灵
			lt->setDisplay(CCSprite::create("Peas.png"));
			// 添加到显示
			this->addChild(lt);
			// 添加到触摸管理，第二个参数，事件 Id
			this->addLsTouch(lt, 100);
	
			LsTouch* lt2 = LsTouch::create();
			lt2->setPosition(ccpAdd(center, ccp(20, 10)));
			lt2->setDisplay(CCSprite::create("Peas.png"));
			addChild(lt2);
			this->addLsTouch(lt2, 101);
	
			bRef = true;
		} while (0);
	
		return bRef;
	}
	
	void TouchEventTest::ccTouchesBegan(CCSet *pTouches, CCEvent *pEvent) {
		CCSetIterator it = pTouches->begin();
		CCTouch* touch = (CCTouch*) (*it);
		// 发送触摸消息，并在 touchEventAction 自动回调相应的事件
		sendTouchMessage(touch);
	}
	
	void TouchEventTest::touchEventAction(LsTouch* touch) {
		CCLog("touch event action id: %d", touch->getEventId());
	}
```	

上述使用方法，在 init() 方法中创建了两个可触摸元素，并设置显示的精灵，这里只实现了 ccTouchesBegan 方法，当然也可以添加 ccTouchesMoved 等方法的实现，这是为了雏形的设计简单，LsTouch 的实现可以自定义，显示什么，范围如何判断可以自行扩展，它本身也是个 CCNode ，所以可以通过 addChild 添加到界面显示，然后调用 addLsTouch 方法，添加到触摸管理，此时 精灵才能在调用 **sendTouchMessage** 时，接受触摸消息，从而判断点击的有效性，并在 touchEventAction 方法自动相应。这里可接受 **复杂多变的界面设计**，应为这并不会影响到触摸消息的管理，它是通过 addLsTouch 方法添加到内部的一个 **CCArray** 之中，如果从界面移除了可触摸元素，可以调用 **removeLsTouch** 方法，自动回收，如果没有显示的调用此方法，将会在基层场景销毁时，自动释放 CCArray 里面的所有元素，区别就是是否能够及时释放元素，但就使用来说，并没什么区别。

简单的使用当然基于简单的设计，请看如下([源码查看](https://github.com/leafsoar/ls-cocos2d-x/tree/master/Learn/Classes/TouchEventTest)，GitHub 之上的源码今后可能有所扩展，而下面贴出的是此时的“雏形”)：

``` c++
	class LsTouchEvent;
	
	/**
	 * 定义可触摸元素，用于统一管理
	 */
	class LsTouch: public CCNode {
	public:
		LsTouch();
		~LsTouch();
		CREATE_FUNC(LsTouch);
		virtual bool init()	;
	
		// 设置显示项
		void setDisplay(CCSprite* dis);
	
		void setEventId(int eventId);
		int getEventId();
	
		/// 常规判断
		bool selfCheck(CCTouch* ccTouch, LsTouchEvent* lsTe);
	
	private:
		// 判断当前的元素是否被点击
		bool containsCCTouchPoint(CCTouch* ccTouch);
		bool isParentAllVisible(LsTouchEvent* lsTe);
	
		// 用户保存显示精灵的 tag
		static const int TAG_DISPLAY = 100;
		int m_iEventId;
	
	};
	
	class LsTouchEvent {
	public:
		LsTouchEvent();
		~LsTouchEvent();
	
		void addLsTouch(LsTouch* touch, int eventId);
	
		void removeLsTouch(LsTouch* touch);
	
		bool sendTouchMessage(CCTouch* ccTouch);
	
		// 返回优先级较高的可触摸对象
		LsTouch* getPriorityTouch(LsTouch* a, LsTouch* b);
	
		virtual void touchEventAction(LsTouch* touch) = 0;
	private:
		CCArray* m_pLsTouches;
	};

	/// 类实现	
	#include "LsTouch.h"
	
	LsTouch::LsTouch() {
		CCLog("LsTouch()");
		m_iEventId = 0;
	}
	
	LsTouch::~LsTouch() {
		CCLog("LsTouch().~()");
	}
	
	bool LsTouch::init() {
	
		return true;
	}
	
	void LsTouch::setDisplay(CCSprite* dis) {
		// 设置之前先清除，没有也无所谓
		removeChildByTag(TAG_DISPLAY, true);
		addChild(dis, 0, TAG_DISPLAY);
	}
	
	void LsTouch::setEventId(int eventId) {
		m_iEventId = eventId;
	}
	
	int LsTouch::getEventId() {
		return m_iEventId;
	}
	
	bool LsTouch::selfCheck(CCTouch* ccTouch, LsTouchEvent* lsTe) {
		bool bRef = false;
		// 可点击项的检测，可扩展
		do {
			// 是否通过点击位置检测
			CC_BREAK_IF(!containsCCTouchPoint(ccTouch));
			// 是否正在运行，排除可能存在已经从界面移除，但是并没有释放的可能
			CC_BREAK_IF(!isRunning());
	
			// 判断是否隐藏
			CC_BREAK_IF(!isVisible());
			// 这里可能还需要判断内部显示项目是否隐藏
			///// 暂留
			// 不仅判断当前元素是否隐藏，还需要判断在它之上的元素直到事件处理层，是否存在隐藏
			CC_BREAK_IF(!isParentAllVisible(lsTe));
	
			bRef = true;
		} while (0);
		return bRef;
	}
	
	bool LsTouch::containsCCTouchPoint(CCTouch* ccTouch) {
		// 获得显示内容
		CCNode* dis = getChildByTag(TAG_DISPLAY);
		CCSprite* sprite = dynamic_cast<CCSprite*>(dis);
		CCPoint point = sprite->convertTouchToNodeSpaceAR(ccTouch);
		CCSize s = sprite->getTexture()->getContentSize();
		CCRect rect = CCRectMake(-s.width / 2, -s.height / 2, s.width, s.height);
		return rect.containsPoint(point);
	}
	
	bool LsTouch::isParentAllVisible(LsTouchEvent* lsTe) {
		bool bRef = true;
		// 向父类转型，以便获取地址比较对象，LsTouchEvent 的对象必须同时直接或者简介继承 CCNode
		CCNode* nLsTe = dynamic_cast<CCNode*>(lsTe);
	
		CCNode* parent = getParent();
		do {
			// 如果遍历完毕，说明 LsTouch 不再 LsTouchEvent 之内
			if (!parent) {
				bRef = false;
				break;
			}
			// 如果 LsTouch 在 LsTouchEvent 之内，返回 true
			// 注意：如果想让LsTouchEvent 处理 不在其 CCNode 结构之内的元素，则取消此处判断
			if (nLsTe == parent) {
				break;
			}
			if (!parent->isVisible()) {
				bRef = false;
				break;
			}
			parent = parent->getParent();
		} while (1);
		return bRef;
	}
	
	LsTouchEvent::LsTouchEvent() {
		CCLog("LsTouchEvent()");
		m_pLsTouches = CCArray::create();
		m_pLsTouches->retain();
	}
	
	LsTouchEvent::~LsTouchEvent() {
		CCLog("LsTouchEvent().~()");
		m_pLsTouches->release();
	}
	
	void LsTouchEvent::addLsTouch(LsTouch* touch, int eventId) {
		touch->setEventId(eventId);
		m_pLsTouches->addObject(touch);
	}
	
	void LsTouchEvent::removeLsTouch(LsTouch* touch) {
		m_pLsTouches->removeObject(touch, true);
	}
	
	bool LsTouchEvent::sendTouchMessage(CCTouch* ccTouch) {
		// 编写判断，集合中的哪个元素级别高，就触发哪一个
		LsTouch* lsTouch = NULL;
	
		// 获得点击的点
		CCObject* pObj = NULL;
		LsTouch* lt = NULL;
		CCARRAY_FOREACH(m_pLsTouches, pObj) {
			lt = dynamic_cast<LsTouch*>(pObj);
			if (lt) {
				if (lt->selfCheck(ccTouch, this)) {
					if (lsTouch == NULL)
						lsTouch = lt;
					else
						// 如果已存在符合条件元素，比较优先级
						lsTouch = getPriorityTouch(lsTouch, lt);
				}
			}
		}
	// 比对最终只有一个元素触发
		if (lsTouch){
			touchEventAction(lsTouch);
			return true;
		}
		return false;
	}
	
	LsTouch* LsTouchEvent::getPriorityTouch(LsTouch* a, LsTouch* b) {
		// 触摸优先级通过 CCNode 树判断，也既是显示层次级别等因素
		// 以当前元素为“根”向父类转型，以便获取地址比较对象，LsTouchEvent 的对象必须同时直接或者简介继承 CCNode
		CCNode* nLsTe = dynamic_cast<CCNode*>(this);
	
		// 共同的分枝
		CCNode* allParent = NULL;
		// 寻找 a 与 b 共同的分枝
		CCNode* nAParent = a;
		CCNode* nBParent = b;
		CCNode* nAChild = NULL;
		CCNode* nBChild = NULL;
		do {
			nAChild = nAParent;
			nAParent = nAParent->getParent();
			if (!nAParent)
				break;
	
			nBParent = b;
			do {
				nBChild = nBParent;
				nBParent = nBParent->getParent();
				if (!nBParent)
					break;
				if (nAParent == nBParent) {
					allParent = nAParent;
					break;
				}
				if (nBParent == nLsTe) {
					break;
				}
			} while (1);
			if (allParent)
				break;
			if (nAParent == nLsTe) {
				break;
			}
		} while (1);
	
		// 此处只需要判断 nAChild 和 nBChild 的优先级即可，默认返回 a
		if (!nAChild || !nBChild)
			return a;
		// 根据 ZOrder 判断，如果 ZOrder一样，根据索引位置判断
		if (nAChild->getZOrder() == nBChild->getZOrder())
			return allParent->getChildren()->indexOfObject(nAChild) > allParent->getChildren()->indexOfObject(nBChild)? a: b;
		else
			return nAChild->getZOrder() > nBChild->getZOrder()? a: b;
	}
```

## 关于后续

实现了这样一个简单的事件处理模型，可以稍加修改扩展，基本能满足大部分的使用需求了，优势是使用简单，当然也有不足之处（这点也是今后需要完善的所在），比如事件的处理统一由场景基层实现调用，而我的理想使用方式，是 LsTouchEvent 可以添加到其它的 LsTouchEvent 之中，并且可以控制这样一种子层的可视范围（这确实很有用处，比如层级遮挡等），这样如果界面太过复杂不用把所有的事件响应都放在场景基层之中了，可以在任意的某一个层处理，分而治之，这样也就能够非常方便的处理非常复杂的 UI 逻辑！而要在现有雏形实现此功能，我们只需要在 LsTouchEvent 内部添加一个 LsTouchEvent 类型的集合，从而使场景基层管理到所有的 LsTouchEvent 事件相应层，LsTouchEvent 将会组成一个树形结构，也可以使触摸消息传递到所有的 LsTouchEvent 层中。如此，场景基层同样能管理到所有的可触摸元素，并判断优先级。

cocos2d-x 本来提供的触摸消息机制，通过实现各个层的 ccTouchesBegan 等方法，使用确实灵活，但界面一复杂，就灵活的有些难以驾驭，比如我们需要在每个地方对内部元素做是否运行（IsRunning）是否隐藏(IsVisible)判断等，还需要对其相应的优先级多做了解，才能保证使用过程中不会出现什么纰漏。

而对于本文，如果有什么异议，或者有什么其它的设计方式，欢迎留言讨论 ~
