---
layout: post
title: Cocos2d-x 内存管理浅说
date: 2013-05-22 23:30
comments: true
categories: Cocos2d-x
---

使用过 Cocos2d-x 都知道，其中有一套自己实现的内存管理机制，不同于一般 C++ 的编写常规，而在使用前，了解其原理是有必要的，网上已经有很多对内部实现详细解说的文章。而对于使用者而言，并不需要对其内部有很深的了解，注重其**“机制”**，而非内部实现，在这里只是简单的聊一聊它的管理方式以及使用，固为浅说。

## 无用对象 与 管理对象

**Cocos2d-x 将会在下一帧自动清理无用的对象，什么是无用的对象，通过 create() 方法创建的就是无用的对象。**

为了简要说明，代码的组织设计一切从简，我们创建了两个辅助类和一个容器类 BaseLayer，在 BaseLayer 之上管理内部对象，并观察它是怎么自动管理对象的。实现了其 构造函数 方法和 析构函数，并做些日志打印，以方便我们观察：

<!-- more -->

``` c++
	class LSLayer: public CCNode {
	public:
		virtual bool init() {
			CCLog("LSLayer().init()");
			return true;
		};
	
		CREATE_FUNC(LSLayer);
	
		LSLayer(){
			CCLog("LSLayer().()");
		};
		~LSLayer() {
			CCLog("LSLayer().~()");
		};
	};
	
	class LSSprite: public CCNode {
	public:
		virtual bool init() {
			CCLog("LSSprite().init()");
			return true;
		};
	
		CREATE_FUNC(LSSprite);
	
		LSSprite(){
			CCLog("LSSprite().()");
		};
		~LSSprite() {
			CCLog("LSSprite().~()");
		};
	};
	
	class BaseLayer: public CCLayer {
	public:
		virtual bool init(){
			CCLog("BaseLayer().init()");
			// 我们创建了两个 “无用”对象
			LSLayer* layer = LSLayer::create();
			LSSprite* sprite = LSSprite::create();
			// 使用了 layer 变为受“管理”的对象
			this->addChild(layer);

			return true;
		};
	
		CREATE_FUNC(BaseLayer);
	
		BaseLayer(){
			CCLog("BaseLayer().()");
		};
		~BaseLayer(){
			CCLog("BaseLayer().~()");
		};
	};
```	
	
如上所示，我们在 BaseLayer 中创建了两个对象， layer 和 sprite，而只使用了 layer ，如果要运行上面的 BaseLayer 代码，我们需要创建一个 BaseLayer  的层对象，并将它添加到运行的场景或者层中： `addChild(BaseLayer::create());`，以保证 BaseLayer 开始运行，现在我们分析一下运行的结果：

``` bash
	// 由 addChild(BaseLayer::create()); 方法开始，创建并初始化了 BaseLayer 层
	cocos2d-x debug info [BaseLayer().()]
	cocos2d-x debug info [BaseLayer().init()]
	// BaseLayer init 方法我们创建了两个对象
	cocos2d-x debug info [LSLayer().()]
	cocos2d-x debug info [LSLayer().init()]
	cocos2d-x debug info [LSSprite().()]
	cocos2d-x debug info [LSSprite().init()]
	// 对象创建完成，紧接着这“无用”对象便已经释放了，而另一个已经使用的对象没有释放
	cocos2d-x debug info [LSSprite().~()]
```	

	
通过上面两个例子对比，对 cocos2d-x 的对象管理有了初步的认识，它会自动清理 **“无用对象”**。为了区分概念，我们将另一种对象称之为 **“管理对象”**，它是受管理的，有用的对象。比如上文中的 **layer**。

！！这也算初步认识，当然，这至少解决了我们这样一个疑问：**我们在场景初始化的时候，通过 create() 创建了成员变量，以备需要的时候使用，但发现在使用的时候这个对象已经不存在了，从而导致程序崩溃。**

### 管理对象不用之时立即回收

我们再继续演变 BaseLayer 的实现，以方便我们观察在每一帧对象的情况，添加实现了定时器功能：

``` c++
	class BaseLayer2: public CCLayer {
	public:
		virtual bool init(){
			CCLog("BaseLayer2().init()");
			// 启用定时器，自动在每一帧调用 update 方法
			this->scheduleUpdate();
			return true;
		};
	
		// 定义 update 统计
		int updateCount;
		LSLayer* layer;
		LSSprite* sprite;
	
		virtual void update(float fDelta){
			// 为了方便观察，不让 update 内部无止境的打印下去
			if (updateCount < 3){
				updateCount ++;
				CCLog("update index: %d", updateCount);
	
				// 在不同的帧做相关操作，以便观察
				if (updateCount == 1){
					layer = LSLayer::create();
					this->addChild(layer);
					sprite = LSSprite::create();
	
				} else if (updateCount == 2){
					this->removeChild(layer, true);
	
				} else if (updateCount == 3){
	
				}
	
				CCLog("update index: %d end", updateCount);
			}
		};
	
		CREATE_FUNC(BaseLayer2);
	
		BaseLayer2():
			updateCount(0),
			layer(NULL),
			sprite(NULL)
		{
			CCLog("BaseLayer2().()");
		};
		~BaseLayer2(){
			CCLog("BaseLayer2().~()");
		};
	};

	// 打印如下
	cocos2d-x debug info [BaseLayer2().()]
	cocos2d-x debug info [BaseLayer2().init()]
	// 第一帧创建两个对象
	cocos2d-x debug info [update index: 1]
	cocos2d-x debug info [LSLayer().()]
	cocos2d-x debug info [LSLayer().init()]
	cocos2d-x debug info [LSSprite().()]
	cocos2d-x debug info [LSSprite().init()]
	cocos2d-x debug info [update index: 1 end]
	// 我们看到 sprite 无用对象在 第一帧和第二帧之间被释放
	cocos2d-x debug info [LSSprite().~()]
	cocos2d-x debug info [update index: 2]
	// 在第二帧移除管理对象，可以看到它是立即释放，在 index: 2 end 之前
	cocos2d-x debug info [LSLayer().~()]
	cocos2d-x debug info [update index: 2 end]
	cocos2d-x debug info [update index: 3]
	cocos2d-x debug info [update index: 3 end]
```	

与无用对象不同的是，管理对象在不用之时，立即释放，这决定着如果想在其它地方使用此对象，在“完全”不用之前，一定要有所作为。重写 update 方法如下：

``` c++
	virtual void update(float fDelta){
		// 为了方便观察，不让 update 内部无止境的打印下去
		if (updateCount < 3){
			updateCount ++;
			CCLog("update index: %d", updateCount);

			// 在不同的帧做相关操作，以便观察
			if (updateCount == 1){
				layer = LSLayer::create();
				this->addChild(layer);
				sprite = LSSprite::create();
				CCLog("%d", layer);
			} else if (updateCount == 2){
				layer->retain();
				this->removeChild(layer, true);
				CCLog("%d", layer);
			} else if (updateCount == 3){
				layer->release();
				if (layer){
					CCLog("%d", layer);
				}
			}

			CCLog("update index: %d end", updateCount);
		}
	};
	
	/// 打印如下
	cocos2d-x debug info [update index: 1]
	cocos2d-x debug info [LSLayer().()]
	cocos2d-x debug info [LSLayer().init()]
	cocos2d-x debug info [LSSprite().()]
	cocos2d-x debug info [LSSprite().init()]
	cocos2d-x debug info [147867424]
	cocos2d-x debug info [update index: 1 end]
	cocos2d-x debug info [LSSprite().~()]
	cocos2d-x debug info [update index: 2]
	// 第二帧并没有释放 layer，因为它还是有用的管理对象
	cocos2d-x debug info [147867424]
	cocos2d-x debug info [update index: 2 end]
	cocos2d-x debug info [update index: 3]
	// 完全弃用，立即释放	
	cocos2d-x debug info [LSLayer().~()]
	// 但是 layer 对象的地址还是可用的
	cocos2d-x debug info [147867424]
	cocos2d-x debug info [update index: 3 end]
```	

**在完全不用之前，要有所作为。** 如果我们将第二帧中的 `layer->retain();` **放在**  `this->removeChild(layer, true);` **之后** 呢，我们知道在 removeChild 之后是立即释放的，此时 layer 对象已经不存在了，而 layer 所指向的内存地址是个无效地址。如果你的程序继续运行，那么一定会出现内存错误。

**如果程序直接错误异常退出，倒也罢了，怕就怕，程序可能继续运行**，layer 虽然是无效地址，但并不是 NULL，可能所指向的地址可用，可能还能继续执行，更可能的还能继续 `layer->retain();` 操作。这会影响我们的判断，程序真的有问题么。如果留下了这种隐患，那么排除错误的难度会大大加深。比如程序莫名其妙的退出，时好时坏！（经过一叶的测试，这种情况是可能发生的，而且频率相当高，测试平台：Linux 平台，Android平台可能性稍低）

第三帧我们通过 `if (layer)` 判断对象是否可用，如果可用我们继续操作 layer ，这样的使用方式也将会留下内存隐患，因为这样的判断是能通过的，但却是 **不一定** 能够正确使用的。

一般而言，我们不一定需要 `if(layer)` 诸如此类的判断，这也是不推荐的。管理对象，**谁使用，那么谁就是可控的！**如果在对象销毁之前 谁 retain() ，那么在 release() 之前，它无需判断即可使用。谁 **addXXX** 使用，一般能通过 **getXXX** 获取。

简而言之，谁使用（引用），你就找谁就行了，不论是获取，或者移除。

我们前面所言，管理对象不用之时，立即回收，那么我们在同一帧使用，然后移除呢？我们继续改写 update 方法，验证想法：

``` c++
	virtual void update(float fDelta){
		// 为了方便观察，不让 update 内部无止境的打印下去
		if (updateCount < 3){
			updateCount ++;
			CCLog("update index: %d", updateCount);

			// 在不同的帧做相关操作，以便观察
			if (updateCount == 1){
				layer = LSLayer::create();
				this->addChild(layer);
				this->removeChild(layer, true);
			} else if (updateCount == 2){

			} else if (updateCount == 3){

			}

			CCLog("update index: %d end", updateCount);
		}
	};

	/// 其打印如下
	cocos2d-x debug info [update index: 1]
	cocos2d-x debug info [LSLayer().()]
	cocos2d-x debug info [LSLayer().init()]
	cocos2d-x debug info [update index: 1 end]
	// layer 在两帧之间释放，也既是在一下帧自动清理
	cocos2d-x debug info [LSLayer().~()]
	cocos2d-x debug info [update index: 2]
	cocos2d-x debug info [update index: 2 end]
```	

这里我们在同一帧 addChild 并且随之 removeChild，那么 layer 的性质又是如何，我们知道 **管理对象** 在不用之时会立即释放，但在这里并没有立即释放，那说明什么，说明 layer 并不是管理对象，还只是无用对象，并且在这一帧结束时，或者说在 **帧过度** 的时候，并没有使用，可想而知，在 帧过度的时候，其内部做了些处理，首先自动清理无用对象，或者将以使用的无用对象变成管理对象，而在以后的帧，如果在管理对象不用之时，将会立即释放。

现在来看一看稍微复杂点的结构会如何。

``` c++
	// 在不同的帧做相关操作，以便观察
	if (updateCount == 1){
		layer = LSLayer::create();
		sprite = LSSprite::create();
		layer->addChild(sprite);
		addChild(layer);
	} else if (updateCount == 2){
	
		this->removeChild(layer, true);
	} else if (updateCount == 3){
	
	}

	/// 打印如下
	cocos2d-x debug info [update index: 2]
	cocos2d-x debug info [LSLayer().~()]
	cocos2d-x debug info [LSSprite().~()]
	cocos2d-x debug info [update index: 2 end]
```	

我们创建了两个对象 layer 和 sprite，将 sprite 添加到 layer，并把通过 addChild(layer) 使用 layer，可以看到，在第二帧移除 layer 的时候，立即释放了 layer 和 sprite 对象。这也是 cocos2d-x 自动管理所实现的功能，**在 使用者 不用的时候，它也将会解除对其它对象的使用。**

基于以上情况，做些变形：

``` c++
	// 在不同的帧做相关操作，以便观察
	if (updateCount == 1){
		layer = LSLayer::create();
		sprite = LSSprite::create();
		layer->addChild(sprite);
	} else if (updateCount == 2){
		this->removeChild(layer, true);
	} else if (updateCount == 3){

	}

	/// 打印如下
	cocos2d-x debug info [update index: 1 end]
	cocos2d-x debug info [LSLayer().~()]
	cocos2d-x debug info [LSSprite().~()]
	cocos2d-x debug info [update index: 2]
	cocos2d-x debug info [update index: 2 end]
```	
	
创建了两个对象 layer 和 sprite，将 sprite 添加到 layer 之中，而对 layer 不做处理，我们知道 layer 在第一帧结束后，会自动释放，所以也会释放其所引用的 sprite，而此时 sprite 的性质就有点微妙了。它在帧过度之间是怎么处理的，它是不是我们这所说的无用对象呢？哈！如果 layer 首先被自动管理，那么它会首先回收，并取消对 sprite 的引用，那么 sprite 就是个无用对象，**被自动回收**。如果 sprite 首先被自动管理，那么它将会先变成一个管理对象，然后在 layer 自动释放并取消对 sprite 引用的时候，**被立即释放**。从效果上来说，都是一帧之内完成的。但具体是哪种情况呢？我不知晓 : p 也不用知晓 ~ 所谓不知为不知，是知也 ～


## 写在后面
自动管理，所谓自动管理就是通过 `create()` 方法创建的对象（当然其内部是通过 autorelease() 方法标示，create 只是提供一个统一的创建对象方式），而什么又是有用无用呢，文中我们看到 `retain()` 和 `release()`，而这就是有用无用的实现原理，使用就 retain ，移除使用就用 release，再细究内部，可知里面维护了一个引用计数，从而判断是否被使用 ，而前文我们知道 layer->addChild(obj)，那么 obj 就为 layer 所用，究其本质，也是其内部调用了其 retain 等方法，可以阅读官方相关文档，有详细的说明，而本文多是以抽象的概念解说其设计理念，从使用者的角度分析在使用过程中可能会出现的问题，因为要想达到相同的自动管理效果，实现方式可以有很多种。别太注重细节，如果有什么疑问，可以像这样，通过几个小例程去验证我们的想法。对于本文，也只是我对 cocos2d-x 自动管理的理解，如果在实现和概念上有什么说的不对，还请指出，毕竟是 **浅说** ～

cocos2d-x 主要以 CCNode 为基类的树形结构组织管理，所以本文所创建的例程，基于 CCNode 编写，当然内存的自动管理还有很多内容，比如缓存的实现，消息机制对象的生命周期等。但基于谁使用，谁处理的原则，思路倒也明晰 ~
