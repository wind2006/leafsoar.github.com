---
layout: post
title: Cocos2d-x 内存管理的一种实现
date: 2013-05-29 10:10
comments: true
categories: Cocos2d-x
---

使用 Cocos2d-x 编写游戏，常被人吐槽，吐槽什么，当然是内存管理，C++ 的使用本以不易，而在 Cocos2d-x 添加了半内存自动管理，在这么一种 **复合机制** 下，使得出现内存问题的概率直线飙升 ~

而关于这一点，可能在于并没有一个通用的合理的内存管理方式，能够更好的使用 Cocos2d-x ，或者说，缺少那么一种 **规范**，如果存在了这么一种 **规范**，而使得 Cocos2d-x 更为简单优雅，那势必是游戏的开发过程，更加从新所欲，把重点放在游戏本身的设计之上。

## Retain 与 Release 乃万恶之源

稍微了解一点就能知道 Cocos2d-x 是基于引用计数来管理内存的，应用计数的加减就是 retain 和 release 方法的实现。在大多数情况下我们不用 **显示** 的去调用这两种方法，如在 **CCNode** 的 **addChild** 和 **removeChild** 方法，**CCArray** 的 **addObject** 和 **removeObject** 等这样成双成对的方法，对于这些的使用很简单，一叶上篇文章 **[Cocos2d-x 内存管理浅说](http://blog.leafsoar.com/archives/2013/05-22-23.html)** 从概念上简单的分析了内部对象的生命周期特点，在此 **浅说** 之中，我刻意的绕过了它的底层实现，并没有深究其原理，对引用计数等概念也只是几句话一带而过，重点放在使用者该关心什么，该注意什么。因为我觉得 **引用计数是个坑，一个很大的坑 ~**

<!-- more -->

当我们想要长期 **持有** 某个对象的时候，我们会用到 retain 和 release 方法，而这种情况我们会经常遇到，如那些 **非CCNode** 类型，比如一个运行场景里面有一个 **CCString** （以CCString 为例，显然此刻你更愿意用 std::string）保存的场景名称，以便我们随时使用，那我们一个简单的做法就是在场景初始化的时候创建 CCString 对象，赋值，然后 retain，在场景结束或者析构函数中 release，这很简单，一个 retain 对应一个 release 就没有问题了，如果问题稍微变的复杂，在程序的运行中，我们可能会改变这个属性值，创建一个新的 CCString 去替换它，那在执行这些操作的时候我们需要很多判断，是否已经有值，首先要解除之前的引用，在重新引用新的对象~~**诸如此类**，如果中间不需要此对象，中间直接释放，那么我们会 **非常华丽的看到在程序代码之中到处穿插着 retain 和 release 操作**。而这些 retain 和 release 虽然成对出现，但不一定在同一个方法，**这就演变成了，所在的不同方法也要成对的调用。**

**你把青蛙放到冷水里，再慢慢地加热，青蛙感觉不到什么，直到最后被烫死。** 使用 retain 和 release 就正如温水里的青蛙，刚开始到也没觉得什么，引用计数概念多好。而到后来，发现越来越难以控制，为时以晚矣～

“如果说C语言给了你足够的绳子吊死自己，那么C++给的绳子除了够你上吊之外，还够绑上你所有的邻居，并提供一艘帆船所需的绳索。”（摘自 **UNIX痛恨者手册**） 而此时 ~~~


## 建立规范 完全消灭 retain 和 release

既然说 retain 和 release 乃万恶之源，那么我们只要 **从源头上，解决这个问题**，如此一切将会变的非常简单，我们将建立一种类似 addChild 这样的 **内部处理** 机制，不用显示的调用 retain 和 release ，从而杜绝了 retain “漫天飞”的可能。而要实现这样的机制，只需简单的设计即可 ~代码实现如下[源码示例](https://github.com/leafsoar/ls-cocos2d-x/blob/master/Learn/Classes/Property/Property.h)：

``` c++
	// 为了方便起见，自定义宏，并且为 varName 的实现加上了 __ls_ 的前缀，前缀可以修改，可以很长很长很长
	// 加 __ls_ 前缀是为了，在使用的过程只通过 set 和 get 属性包装器调用，而不要直接使用此属性
	#define LS_PRE(p) __ls_##p
	//#define LS_PRE(p) __retain_##p			// 其它前缀都行，目的是为了不让在直接使用此类型对象

	//	此处定义以弃用
	//	#define LS_PROPERTY_RETAIN(varType, varName, funName)\
	//	private: varType LS_PRE(varName);\
	//	public: void set##funName(varType value){\
	//		CC_SAFE_RELEASE_NULL(LS_PRE(varName));\
	//		LS_PRE(varName) = value;\
	//		CC_SAFE_RETAIN(LS_PRE(varName));\
	//	}; \
	//	public: varType get##funName(){return LS_PRE(varName);};

	// 经朋友提醒，发现 cocos2d-x 已经实现了相应功能的宏，并且更好用，那这里的二次包装就算是仅仅加个前缀吧 ！！！
	#define LS_PROPERTY_RETAIN(varType, varName, funName)\
		CC_SYNTHESIZE_RETAIN(varType, LS_PRE(varName), funName);

	// 初始化和释放包装宏，主要为了封装前缀，始定义统一
	#define LS_P_INIT(p) LS_PRE(p)(0)
	#define LS_P_RELEASE(p) CC_SAFE_RELEASE_NULL(LS_PRE(p))
	
	/**
	 * 自定义类型数据：用户信息
	 */
	class LUser: public cocos2d::CCObject{
	public:
		CREATE_FUNC(LUser);
		virtual bool init(){
			return true;
		};
		LUser(){
			CCLog("LUser()");
		};
		~LUser(){
			CCLog("LUser().~():%s", m_sUserName.c_str());
		};
	
		std::string m_sUserName;		// 用户名
		std::string m_sPassword;		// 用户密码
	};
	
	class PropertyTest: public CCLayer{
	public:
		CREATE_FUNC(PropertyTest);
	
		virtual bool init(){
			CCLog("PropertyTest().init()");
			LUser* lu = LUser::create();
			lu->m_sUserName = "leafsoar";
			lu->m_sPassword = "123456";
			setLUser(lu);
	
			// 为了方便在不同帧测试，启用定时器
			this->scheduleUpdate();
	
			return true;
		};
	
		virtual void update(float fDelta){
		        // 为了方便观察，不让 update 内部无止境的打印下去
		        if (updateCount < 5){
		            updateCount ++;
		            CCLog("update index: %d", updateCount);
		            // 在不同的帧做相关操作，以便观察
		            if (updateCount == 1){
		            	// 这里使用 getLUser 获取数据，而非 [__ls_]m_pLUser，所以我设置了前缀
		            	if (getLUser())
		            		CCLog("log lu: %s", getLUser()->m_sUserName.c_str());
	
		            } else if (updateCount == 2){
		            	// 重新赋值
		            	LUser* lu = LUser::create();
		            	lu->m_sUserName = "一叶";
		            	setLUser(lu);
		            } else if (updateCount == 3){
		            	if (getLUser())
		            		CCLog("log lu: %s", getLUser()->m_sUserName.c_str());
		            } else if (updateCount == 4){
		            	// 这里调用 seLUser(0),直接取消引用持有对象，如果不调用也没有关系
		            	// 因为在当前类析构的时候会自动检测释放
		            	setLUser(0);
		            }
		            CCLog("update index: %d end", updateCount);
		        }
		    };
	
		// 构造函数，初始化 LS_PROPERTY_RETAIN 属性为空
		PropertyTest():
			LS_P_INIT(m_pLUser),
			updateCount(0)
		{
		};
	
		// 析构函数释放
		~PropertyTest(){
			LS_P_RELEASE(m_pLUser);
		};
	
		// 使用 LS_PROPERTY_RETAIN 宏定义的属性，必须在构造和析构函数中初始化和释放
		// 初始化为 0 或者 NULL，是为了在进行赋值操作前判断是否以有引用
		// 析构函数释放是为了解除对持有对象的引用，如果有的话
		LS_PROPERTY_RETAIN(LUser*, m_pLUser, LUser);
	
	private:
		int updateCount;
	};

	/// 程序执行打印如下
	cocos2d-x debug info [PropertyTest().init()]
	// init 方法创建对象并通过 setLUser 持有对象
	cocos2d-x debug info [LUser()]
	cocos2d-x debug info [update index: 1]
	// 第一帧顺利访问 持有对象
	cocos2d-x debug info [log lu: leafsoar]
	cocos2d-x debug info [update index: 1 end]
	cocos2d-x debug info [update index: 2]
	// 第二帧创建新的 用户信息
	cocos2d-x debug info [LUser()]
	// 通过 setLUser 改变用户信息，这会使得之前设置的用户信息“自动”释放
	cocos2d-x debug info [LUser().~():leafsoar]
	cocos2d-x debug info [update index: 2 end]
	cocos2d-x debug info [update index: 3]
	// 跨帧继续访问新值
	cocos2d-x debug info [log lu: 一叶]
	cocos2d-x debug info [update index: 3 end]
	cocos2d-x debug info [update index: 4]
	// 调用了 setLUser(0) 说明已经解除了之前持有对象的引用，如果有的话
	cocos2d-x debug info [LUser().~():一叶]
	cocos2d-x debug info [update index: 4 end]
	cocos2d-x debug info [update index: 5]
	cocos2d-x debug info [update index: 5 end]
```	

通过上面的例子，可以看到将 **持有对象** 的操作变的非常简单，**只通过** set 和 get 属性包装器存取数据，而并没有 **显示** 的调用 retain 和 release 方法来操作，最大程度的自动化管理引用计数问题，一切皆在掌控之中。从此，世界清净了 ~ **你不用再为何时 retain 何处 release 而烦恼。**

而要做到如上的使用方法，在定义之初需规范化设计，大致如下：

* 通过 **LS_PROPERTY_RETAIN** 宏创建 **可持有对象属性**，并自动创建 set 和 get 属性包装器。宏的设计并非毫无来由，我们知道 cocos2d-x 内部定义了很多以 **CC_** 为前缀的宏，方便使用，比如 **CC_PROPERTY[xxx]** 此类。set 方法会自动的根据需要处理 retain 和 release。
* 宿主类的构造函数必须初始化对象为 NULL 或者 0，这是 C++ 的特性使然。LS_P_INIT，简化了操作。
* 宿主类的析构函数必须释放对象[如果有]，这样我们就不用 **显示** 的调用释放了。可以通过 LS_P_RELEASE 调用。

### LS_PROPERTY_RETAIN 宏的实现

在上面的例程中，我们使用了 **LS_PROPERTY_RETAIN(LUser*, m_pLUser, LUser);** 定义一个属性，那么我们看这个宏做了哪些事情，我们展开这个宏看看：

``` c++
	LS_PROPERTY_RETAIN(LUser*, m_pLUser, LUser);
	// 展开如下
	private:
		// 定义私有属性
		LUser* __ls_m_pLUser;
	public:
		// 实现 set 方法
		void setLUser(LUser* var){
			// 首先释放当前的持有对象，没有则罢，如果有，那么就 release，因为如果有值，毕定是通过此方法设置并 retain 的
			if (__ls_m_pLUser != var){
				// 持有新的对象，这些都是 SAFE  安全操作的
				CC_SAFE_RETAIN(var);
				// 这里是 cocos2d-x 提供的宏，就不展开了				
				CC_SAFE_RELEASE(__ls_m_pLUser);
				// 设置新的属性
				__ls_m_pLUser = var;
			}
		}; 
	public:
		LUser*  getLUser(){
			// 直接返回持有对象
			return __ls_m_pLUser;
		};
```		
	
基本在设计之时，满足以上规范，就能想这里一样，通过 set 和 get 简单的对可持有对象进行任意的操作了。

## 应用

这样的设计使得 **所有基于** CCObject 的类型都能够方便的使用。那我们就能够很容易的持有 CCNode，层，精灵，CCArray，等数据了。而且不会看到漫天飞舞的 retain 和 release ~

当然作用还不止如此，我们可能创建自己的类型继承 CCObject 以方便统一管理，在配合 CCArray ，使自定义的数据和 cocos2d-x **无缝的集成**。有些游戏需要处理很多数据，如网络传输接受的数据，自定义常用数据等 ~

文中我们自定义了 LUser 是继承于 CCObject  的，这只是简单数据类型，复杂点的，LUser 中包含了其它 CCObject 的数据，如果按照以前的写法，设置之后就 retain ，那很难判断在哪里 release。如下：

``` c++
	class LUser: public cocos2d::CCObject{
	public:
		CREATE_FUNC(LUser);
		virtual bool init(){
			return true;
		};
		LUser(){
			CCLog("LUser()");
		};
		~LUser(){
			CCLog("LUser().~():%s", m_sUserName.c_str());
		};
	
		std::string m_sUserName;		// 用户名
		std::string m_sPassword;		// 用户密码
	
		// 其它数据
		CCSprite* m_pSprite;
	};

	// 那么我们的使用过程
	LUser* lu = LUser::create();
	lu->m_sSprite = CCSprite::create("a.png");
	// 如果这里不 retain  则以后就用不到了
	lu->m_sSprite->retain();
```	

LUser 持有 m_sSprite 正如 文中 PropertyTest 持有 m_pLUser 一样，我们重新设计：

``` c++
	class LUser: public cocos2d::CCObject{
	public:
		CREATE_FUNC(LUser);
		virtual bool init(){
			return true;
		};
		LUser():
			LS_P_INIT(m_pSprite)
		{
			CCLog("LUser()");
		};
		~LUser(){
			CCLog("LUser().~():%s", m_sUserName.c_str());
			LS_P_RELEASE(m_pSprite);
		};
	
		std::string m_sUserName;		// 用户名
		std::string m_sPassword;		// 用户密码
	
		// 其它数据
		LS_PROPERTY_RETAIN(CCSprite*, m_pSprite, Sprite);
	
	};

	// 使用方法
	LUser* lu = LUser::create();
	lu->m_sUserName = "一叶";
	// 这里的 sprite 会随着 lu 的消亡而消亡，不用管释放问题了
	lu->setSprite(CCSprite::create("a.png"));
```	

这样便将 m_pSprite 控制权，完全交给了 LUser 来处理了。基于这样的考虑，我们完全可以使用复杂的自定义类型，包含很多 CCObject 属性，而属性之中可能又包含其它 CCObject 的类型，而并不用担心释放问题，**谁持有，谁管理，谁释放**(而不会出现 lu->m_sSprite->retain(); 这样的情况)。这些数据可以在游戏中任意的传递，并且都是CCObject 类型的，并很好的结合 CCArray 管理。让自定义类型与 cocos2d-x 两者天衣无缝，配合无间 ~

这里自定义的宏，加了个复杂的前缀，仅仅想提醒大家，只通过 set 和 get 来进行存取的操作，从而避免使用 retain 和 release 来管理，更简单的写法，使用 cocos2d-x 自带的宏即可：

``` c++
	//  定义可以加 "__" 双下划，以告诉自己这是可持有属性
	CC_SYNTHESIZE_RETAIN(LUser*, __m_pLUser, LUser);

	// 构造函数直接使用 __m_pLUser(0)

	// 析构函数调用如下
	CC_SAFE_RELEASE_NULL(__m_pLUser);

	// 如此倒也省事，事省 : P
```	

### 为什么 LUser 继承自 CCObject
如果不集成自 CCObject 而使用原来的 C++ 方式也并无不可，但 CCObject 的优势是很明显的，如果能够善于使用。如果你想在 cocos2d-x 一个CCNode绑定数据有 setUserObject() 方法，如果多个 LUser 那么可以用 CCArray 进行管理，如果你想使用通知功能 CCNotificationCenter，而此  LUser 是可被传递的，我们设置了 LUser 然后靠诉别人我更新了，发送一条通知，谁对这个通知感兴趣，那谁就自己处理去吧 ~ 如果 ~ 如果你对此文感兴趣，不妨一试 ~
