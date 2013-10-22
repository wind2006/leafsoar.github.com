---
layout: post
title: Cocos2d-x 屏幕适配新解
date: 2013-05-10 19:02
comments: true
categories: Cocos2d-x
---

为了适应移动终端的各种分辨率大小，各种屏幕宽高比，在 cocos2d-x（当前稳定版：**2.0.4**） 中，提供了相应的解决方案，以方便我们在设计游戏时，能够更好的适应不同的环境。

而在设计游戏之初，决定着我们屏幕适配的因素有哪些，简而言之只有两点：**屏幕大小 和 宽高比**。这两个因素是如何影响游戏的：

* **屏幕大小：** 从小分辨率 **480x320** 到 **1280x800** 分辨率，再到全高清 **1080p**，从手机到平板，还有苹果设备的 **Retina** 屏，这么多不同的分辨率，而且大小差距甚大，不可能做到一套资源走天下，资源往小了设计，在大屏幕会显示模糊，图片往大了设计，在小屏幕设备又太浪费，而且小屏幕的手机硬件资源也会相对的紧缺，所以 **根据屏幕大小使用不同的资源** 是有必要的，而 cocos2d-x 也帮我们解决了这一点。
* **宽高比：** 什么是宽高比，就是你的屏幕是方的还是长的，靠近方形的分辨率如 480x320，比例为 **3:2**，还有 960x540 的 **16:9** 标准宽屏，这也算是两种总极端情况了，如果能在这两种比例情况做好适配基本就可以了，如果比 3:2 “更方”如 4:3，比 16:9 “更长”，那么不论如何布局，显示效果差距甚大，最好对固定比例优化吧。当在宽高比在一定范围内，可以通过灵活编写程序去适应，而在显示效果上，cocos2d-x 为我们提供了三种模式，这些 **模式更多的是帮我们解决比例不一的情况而存在** 的，如果只是屏幕大小（比例一样），那通过简单的放大缩小即可完成。

<!-- more -->

## 三种模式

说是三种模式，其实还有一种 **无模式**，也就是 cocos2d-x 默认的适配方案，现在我们就来认识一下这些模式，并且通过这些模式去认识其中一些概念 **FrameSize**、**WinSize**、**VisibleSize**、**VisibleOrigin**，以及它们存在的意义，并且最后灵活运行这些概念 **创建出一个不属于这些模式而超越这些模式的新适配解决方案**，这是最终目的。

### kResolutionUnKnown 认识 FrameSize
这是 cocos2d-x 编写的默认模式，没有做任何处理，在这种情况下，游戏画面的大小与比例都是不可控的，在程序运行之初，由各个平台入口函数定义画面大小：

``` c++
	// proj.linux/main.cpp  linux 平台手动指定画面大小
	CCEGLView* eglView = CCEGLView::sharedOpenGLView();
    eglView->setFrameSize(720, 480);

	// proj.android/jni/hellocpp/main.cpp android 平台由 jni 调用传入设备分辨率参数
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
	        // other
			...
	    }
	}
```	

在此我们首先认识了 **FrameSize** 参数，在游戏运行时，我们可以通过 `CCEGLView::sharedOpenGLView()->getFrameSize();` 获得此值。如果在手机上运行，那么不同分辨率将会得到不同的值，既然这个值不可控，那么在写游戏中也就没有参考价值了，比如我们写一个精灵的位置距离底部 320 高度，在 480x320 分辨率，能看到其在屏幕上方，如果换一台手机分辨率 960x540 那么只能显示在中间靠上的位置，如果设置精灵位置为距离屏幕上方（高度）320，反之依然，显示效果不一。

此时可行的方案是使用百分比，如精灵位置在屏幕横向距离左边 1/3 宽度，在 1/2 正中间处，而类似这样的设置也不用依赖 FrameSize 的具体数值。而这样的做法，使得内部元素像弹簧一样，随着 FrameSize 的大小改变而改变，伸缩或者挤压，对于图片资源大小也是完全不可控，如果根据屏幕大小放大缩小，那我们可以考虑用下面要说的模式，在此不推荐使用  cocos2d-x 的无模式方案。

	
### kResolutionExactFit and kResolutionShowAll 认识 WinSize

在 AppDelegate.cpp 处可以通过设置：

``` c++
	CCEGLView::sharedOpenGLView()->setDesignResolutionSize(720, 480, kResolutionShowAll);
	// 或者
	CCEGLView::sharedOpenGLView()->setDesignResolutionSize(720, 480, kResolutionExactFit);
```	

DesignResolutionSize！顾名思义，也就是逻辑上的游戏屏幕大小，在这里我们设置了其分辨率为 720x480 为例，那么在游戏中，我么设置精灵的位置便可以参照此值，如 左下角 ccp(0,0)，右上角 ccp(720, 480)，而不论 FrameSize 的大小为多少，是 720x480 也好，是 480x320 也罢，总能正确显示其位置，左下角和右上角。能够实现这一点的原因是，固定了设计分辨率大小，从而确定了其固定的宽高比，它的 **优势** 是可以使用具体的数值摆放精灵位置，**不会因为实际屏幕大小宽高比而是内部元素相对位置关系出现混乱**。

而为了保持画面的宽高比，cocos2d-x 做了些牺牲，牺牲了什么呢？kResolutionExactFit 牺牲了画质而保持了全屏显示，对画面进行了拉伸，这意味着什么？意味着相对极端情况下，本来精灵是方形的，显示出来变成长方形，本来圆形的变成了椭圆，固此模式不推荐使用。kResolutionShowAll 为了保持设计画面比例对四周进行留黑边处理，使得不同比例下画面不能全屏。鱼和熊掌不能兼得也 ~

我们可以通过如下方法获取到 **setDesignResolutionSize** 所设置的值：

``` c++
	CCSize winSize = CCDirector::sharedDirector()->getWinSize();
```	

我们可以用 [Cocos2d-x 程序是如何开始运行与结束的](http://blog.leafsoar.com/archives/2013/05-05-23.html) 一文的方法，跟踪 WinSize 的初始化，获取过程，在这里简单提一下，如下步骤：

``` c++
	// 获得 winSize
	CCSize winSize = CCDirector::sharedDirector()->getWinSize();

	// 查看其 getWinSize(); 方法实现
	[cocos2dx-path]/cocos2dx/CCDirector.cpp
	
	CCSize CCDirector::getWinSize(void)
	{
    	return m_obWinSizeInPoints;
	}

	// 而 m_obWinSizeInPoints 是何时被赋值的
	[cocos2dx-path]/cocos2dx/platform/CCEGLViewProtocol.cpp

	void CCEGLViewProtocol::setDesignResolutionSize(float width, float height, ResolutionPolicy resolutionPolicy)
	{
		...
		...
	    m_obDesignResolutionSize.setSize(width, height);
	    
		...
		...
	    CCDirector::sharedDirector()->m_obWinSizeInPoints = getDesignResolutionSize();
	}
	
	const CCSize& CCEGLViewProtocol::getDesignResolutionSize() const 
	{
    	return m_obDesignResolutionSize;
	}
```

具体的优势：通过设置逻辑分辨率大小，相比无模式，可以帮我们解决了屏幕自动放大缩小问题，并且保持屏幕宽高比，使得游戏更好设计，可以将设计画面大小作为默认背景图片大小等，唯一点遗憾就是那点前面所提到的一点点牺牲。

kResolutionShowAll 方案可以作为我们的默认解决方案，使得游戏的设计更为简化，但为了补填拉伸或留黑边这点缺憾，进入下一个模式！

### kResolutionNoBorder 了解 VisibleSize 与 VisibleOrigin

此模式可以解决两个问题，其一：游戏画面全屏；其二：保持设置游戏时的宽高比例，相比 kResolutionShowAll 有所区别的是，**为了填补留下的黑边，将画面稍微放大，以至于能够正好补齐黑边**，而这样做的后果可想而知，补齐黑边的同时，另一个方向上将会有一部分画面露出屏幕之外，如下示意图：

![图片](/images/2013/screen-resolution-1.jpg)

黑色边框标示实际的屏幕分辨率，紫色区域标示游戏设计大小，而通过放大缩小，保持宽高比固定，
可以看到 **Show All** 之中的黑色阴影部分为留边，而 **No Border** 的紫色阴影部分则不能显示，而这紫色区域的大小是游戏设计之时是不可控的。那么原设计的画面大小就失去了 **一定的** 参考价值了，因为这可能让你的画面显示残缺。这时仅仅通过 WinSize 满足不了我们的设计需求，所以引入了 **VisibleSize** 与 **VisibleOrigin** 概念。

![图片](/images/2013/screen-resolution-2.png)

如上所示，紫色区域是被屏幕截去的部分，不可显示的，根据实际情况，可能出现横向截取和竖向截取，这取决于实际分辨率的宽高比。而 A、B、C、D所标示的是设计分辨率，固定大小。如果我们想让一个精灵元素显示在屏幕上方靠边，那么如果使用 WinSize 的高度设置其位置，可能出现的情况就是显示到屏幕之外了。FrameSize 和 WinSize 我们已经知道其概念，而 VisibleSize 和 VisibleOrigin 所代表的是什么呢，又时如何为我们解决靠边的问题！注意上图下方的定义， **VisibleSize = H I J K 是用紫色标注的。** 而在上图是 **黑色** 标注，标示屏幕实际分辨率，虽然 FrameSize 和 VisibleSize 都是 H I J K，但其意义不同，紫色表明它是与设计分辨率相关的。

FrameSize 是实际的屏幕分辨率，而 **VisibleSize 是在 WinSize 之内，保持 FrameSize 的宽高比所能占用的最大区域**，实际屏幕分辨率 H I J K (黑色) 可以大于 WinSize ，但VisibleSize 一定会小于或者等于 WinSize，这两者相同的是宽高比。VisibleSize 有着 WinSize 大小（随WinSize  的大小改变而改变），还有着 FrameSize 的宽高比，它标示 **在设计分辨率（WinSize）下，在屏幕中的可见区域大小。** 而 VisibleOrigin 则标示在设计分辨率下被截取的区域大小，用点 K 标示，有了这些数据，我们想让游戏元素始终在屏幕显示的区域之内不成难事。下面通过几个数值带入，加深这些概念的印象。

``` c++
	// 组[1] :
	FrameSize: 			width = 720, height = 420
	WinSize: 		   	width = 720, height = 480
	VisibleSize:		width = 720, height = 420
	VisibleOrigin:		x = 0, y = 30

	// 组[2] :相比 组 [1] FrameSize 不变 VisibleSize 和 VisibleOrigin 随着 WinSize 的变小而变小
	FrameSize: 			width = 720, height = 420
	WinSize: 		   	width = 480, height = 320
	VisibleSize:		width = 480, height = 280
	VisibleOrigin:		x = 0, y = 20

	// 组[3] : 相比组 [1] WinSize 不变，VisibleSize 随着 FrameSize 的比例改变而改变
	FrameSize: 			width = 720, height = 540
	WinSize: 		   	width = 720, height = 480
	VisibleSize:		width = 640, height = 480
	VisibleOrigin:		x = 40, y = 0

	// WinSize VisibleSize VisibleOrigin 与都设计的分辨率相关，满足如下关系
	WinSize.width = (VisibleOrigin.x * 2) + VisibleSize.width
	WinSize.height = (VisibleOrigin.y * 2) + VisibleSize.height
```	

NoBorder 具体的使用方法可以参考 cocos2d-x 自带例程 **TestCpp** ，有详细的使用方法，并且封装了 **VisibleRect** 类，可以获取设计分辨率，不同比例屏幕之时的主要参考点，屏幕四个拐角，和边的中点等，让我们设置元素位置时，使其总能显示在屏幕之内，这里就不详细介绍了。

基于这几种模式的程序使用方法，cocos2d-x  自带例程或者网上有很多教程，这里只详细解释了其中各种概念，而知道了这些概念，当然用起来就没有多大问题了。

## **kResolutionLeafsoar** 
！！！这是什么模式！好吧，Leafsoar 是 一叶 的 ID ，或者是本博客的一级域名而已 :P  在 cocos2d-x 中并没有这种模式。除却 **UnKnown** 与 **ExactFit** 不说，ShowAll 的优势是，只需要一个设计分辨率，然后通过 WinSize 设置相对对位即可，而且位置的最大长宽都是确定，方便了开发，但屏幕不能填满， NoBorder 模式的优势是在画面不变形的情况下，实现全屏，显示效果更好，但 WinSize 一定程度失效，需要通过运行时计算 VisibleSize 和 VisibleOrigin 来设置位置，由于是运行时计算，所以也就会出现，各种屏幕显示效果不一样的情况。

ShowAll 和 NoBorder 各有所长，各有所短，而这里提出的新适配解决方案正是取两者之长，舍两者之短的 **组合模式**。简单说来就是用 NoBorder 去实现 ShowAll 的思想。NoBorder 可以保证全屏利用，ShowAll 可以更好的使用实际设计坐标固定位置，而且相对位置不会随宽高比的改变而改变，这在编写游戏的时候能方便不少。先上一个示意图，一目了然 （两个图，两个方向）：

![图片](/images/2013/screen-resolution-3.png)

在原来 NoBorder 模式示意图上添加了新的概念，**LsSize = X Y M N** (leafsoar 简写了，为了不跟 cocos2d-x 的一些概念混淆，什么名字不重要，只要了解其含义即可)，在 NoBorder 模式下的 LsSize 相对于 FrameSize 而言，正如 在 **ShowAll** 模式下的 WinSize 相对于 FrameSize，所以说这是 ShowAll NoBorder 的组合概念，**而这里的 LsSize 与 WinSize 的宽高比是一致的。**

猛地一看，似乎把问题复杂化了，仔细一看，还不如猛地一看 ~~

在 ShowAll 中，WinSize 作为最高的宽高，以此参照设置位置，因为在此范围内都能在屏幕上显示，用了 NoBorder 使得四周可能被截去一块区域，而这个区域大小不可控制，所以不能再使用 WinSize 作为参考点来设置位置，而这里的 LsSize 同样，因为 LsSzie 不论在什么情况下，总能显示在屏幕之内，我们可以方便的使用 LsSize  作为坐标系参考，并且可以全屏显示，在配合 VisibleSize ，相比纯的 NoBorder 加强了不少。它可以怎么用？

可以把 LsSize 当作 ShowAll 中的 WinSize 来用，而黑边可以使用稍大的图片填充，或者使用其它**图片修饰边框**，修饰的边框图案可大可小，可长可短，填充屏幕，保持全屏。


## 开始基于 LsSize 的游戏设计实现

为了能够准确实现基于 LsSize 的设计，初步计划将 LsSize 设定在 480x320 的分辨率方案，为此做了些准备，首先不使用任何模式情况下，在场景内调用如下：

``` c++
	CCSize size = CCDirector::sharedDirector()->getWinSize();

	CCPoint center = ccp(size.width/2, size.height/2);

	// 大小 600x500 为了 NoBorder 看到效果，使用稍大的背景图
	CCSprite* pb = CCSprite::create("Back.jpg");
	pb->setPosition(center);
	this->addChild(pb, 0);

	// 480x320 此图为使用于设计分辨率 LsSize 的图片
	CCSprite* pSprite = CCSprite::create("HelloWorld.png");
	pSprite->setPosition(center);
	this->addChild(pSprite, 0);

	// 37x37 在 480x320 画面的四个拐角处，添加参照
	CCSprite* p1 = CCSprite::create("Peas.png");
	p1->setPosition(ccpAdd(center, ccp(-240, -160)));
	this->addChild(p1);

	CCSprite* p2 = CCSprite::create("Peas.png");
	p2->setPosition(ccpAdd(center, ccp(240, 160)));
	this->addChild(p2);

	CCSprite* p3 = CCSprite::create("Peas.png");
	p3->setPosition(ccpAdd(center, ccp(-240, 160)));
	this->addChild(p3);

	CCSprite* p4 = CCSprite::create("Peas.png");
	p4->setPosition(ccpAdd(center, ccp(240, -160)));
	this->addChild(p4);
```	

显示效果：(FrameSize = 640x540)
![图片](/images/2013/screen-resolution-4.jpg)

显示效果：(ShowAll; FrameSize = 520x320; WinSize = 480x320)
![图片](/images/2013/screen-resolution-5.jpg)

显示效果：(NoBorder; FrameSize = 520x320; WinSize = 480x320)
![图片](/images/2013/screen-resolution-6.jpg)

通过效果我们可以看到，在相同 FrameSize 下 NoBorder 时，画面由于填充了黑边，将画面放大，以至于上下有部分显示不全，通过拐角四个精灵可以看出。

好！既然我们知道是由于放大所致，那么我们将画面缩小呢？cocos2d-x 提供了一个方法，我们调用如下代码：

``` c++
	CCDirector *pDirector = CCDirector::sharedDirector();
	pDirector->setContentScaleFactor(
					CCEGLView::sharedOpenGLView()->getScaleY() );
```					
			
为了弥补画面因需要不填空白出现的方法，我们将画面缩小，放大系数可以通过 **CCEGLView::sharedOpenGLView()->getScaleY() ** 取得。其实 setContentScaleFactor 方法是为了适配不同资源而设计的，可以用此方法对不同资源适配，缩放等。效果如下：

![图片](/images/2013/screen-resolution-7.jpg)

我们看到 480x320 的图片显示完全正确了，也正是我们想要的效果，但唯一的缺点是 ~~ 拐角处四个精灵的位置依然不是我们想要的，我们设计的位置是以 480x320 设置位置的，而 WinSize 也是 480x320 ，而此时基于 480x320 的设计必然会显示到屏幕之外，而要想不修改精灵位置，而让其显示正确的位置，那么为了保证 LsSize 的固定，我们需要一个方法，那就是 **动态设置 WinSize**。

什么意思？我们知道一般这些模式设计游戏时，是通过 **setDesignResolutionSize** 设置 WinSize 的，这个值在游戏运行其间是定植，动态改变的是 VisibleSize 等，而这里提出了 LsSize 的概念，可想而知，如果 WinSize 固定，那么 LsSize 会随着屏幕宽高比的改变而改变，那么我们反其道而行，固定 **LsSize** 值，那么在运行时可以通过实际的宽高比来算得 WinSize 的值，这样动态算得的 WinSize 值就能够保证我们的 LsSize 是一个定值了。

**相对论**，WinSize 与 LsSize 的值是相对的，与其通过固定 WinSize 在运行时动态获得 LsSize （这也是 NoBorder 的默认方式，而导致的结果是 WinSize 没有参考价值），不如我们固定 LsSize 而在运行时算得 WinSize 设置来的要更妙一些。

现在不使用 **setContentScaleFactor** 方法，而修改 **setDesignResolutionSize** 这里的值，我们知道 WinSize 是 480x320 时，LsSize 必然会小于此值，而 NoBorder 的放大系数我们可以通过如下方式算得（可以参考setDesignResolutionSize方法内部实现），并在 AppDelegate 里执行：

``` c++
	CCSize frameSize = CCEGLView::sharedOpenGLView()->getFrameSize();
	// 设置 LsSize 固定值
	CCSize lsSize = CCSizeMake(480, 320);

	float scaleX = (float) frameSize.width / lsSize.width;
	float scaleY = (float) frameSize.height / lsSize.height;

	// 定义 scale 变量
	float scale = 0.0f; // MAX(scaleX, scaleY);
	if (scaleX > scaleY) {
		// 如果是 X 方向偏大，那么 scaleX 需要除以一个放大系数，放大系数可以由枞方向获取，
		// 因为此时 FrameSize 和 LsSize 的上下边是重叠的
		scale = scaleX / (frameSize.height / (float) lsSize.height);
	} else {
		scale = scaleY / (frameSize.width / (float) lsSize.width);
	}

	CCLog("x: %f; y: %f; scale: %f", scaleX, scaleY, scale);

	// 根据 LsSize 和屏幕宽高比动态设定 WinSize
	CCEGLView::sharedOpenGLView()->setDesignResolutionSize(lsSize.width * scale,
			lsSize.height * scale, kResolutionNoBorder);
```			

显示效果：（NoBorder 模式 ;FrameSize = 520x320; LsSize = 480x320; WinSize = 动态获取）
![图片](/images/2013/screen-resolution-8.jpg)

我们看到在没有修改源代码，并且在设计中使用 480x320 的参考系，也既是基于  LsSize 的设计显示效果如我们预期，那么我们换一个 FrameSize 来看看是否能够自动适应呢？如下：

显示效果：（NoBorder 模式 ;FrameSize = 600x480; LsSize = 480x320; WinSize = 动态获取）
![图片](/images/2013/screen-resolution-9.jpg)

到此，基于 LsSize 参考系的游戏设计已经完成了，这样做的好处是很明显的，集 ShowAll 和 NoBorder 的优点于一处，这里的图片元素是为了好定位，实现的需要而写的，具体场景可以使用背景地图，或一张大的图片显示，而没有任何影响，也可以继续使用 VisibleSize 得到 LsSize 之外的部分区域大小，在 LsSize 之外可以使用背景图片作为装饰，即保证了游戏的全屏，又保证了游戏设计时的方便，如果使用完全基于 LsSize  的设计实现，除了显示背景装饰之外，我们不想让 LsSize 的内部元素显示到 LsSize 之外如何做呢？我们只需要设定 LsSize 层的的显示区域即可，我们可以修改场景的实现：

``` c++
	// 这里先简单实现思路
	
	CCScene* HelloWorld::scene() {
	
		CCScene *scene = CCScene::create();
		// 创建背景层
		CCLayer* b = CCLayer::create();
		scene->addChild(b);

		// 添加背景图片和设置位置，可以使用其它装饰，或者小图片屏幕都行
		CCSize size = CCDirector::sharedDirector()->getWinSize();
		CCPoint center = ccp(size.width/2, size.height/2);
		CCSprite* pb = CCSprite::create("Back.jpg");
		pb->setPosition(center);
		b->addChild(pb, 0);

		// 创建 LsLayer 层
		HelloWorld *lsLayer = HelloWorld::create();
		scene->addChild(lsLayer);
	
		return scene;
	}

	// 在 HelloWorld 中重写 visit() 函数 设定显示区域
	void HelloWorld::visit() {
		glEnable(GL_SCISSOR_TEST);              // 开启显示指定区域
		// 在这里只写上固定值，在特性环境下，以便快速看效果，实际的值，需要根据实际情况算得
		glScissor(20, 0, 480, 320);     // 只显示当前窗口的区域
		CCLayer::visit();                       // 调用下面的方法
		glDisable(GL_SCISSOR_TEST);             // 禁用
	}
```	

显示效果：（NoBorder 模式 ;FrameSize = 520x320; LsSize = 480x320; WinSize = 动态获取）
![图片](/images/2013/screen-resolution-10.jpg)

## 屏幕适配新解
看完这篇文章想必对 cocos2d-x 的屏幕适配方案及其原理有了相当的认识，从内部提供的三种模式，再到我们自定义基于 LsSize 的 **Leafsoar 模式** (好把，因该叫做 ShowAllNoBorder)。这里已经给出了完全的实现原理以及实现方法，并配有效果图，当然这其中还有些细节需要注意，比如我们基于 LsSize 的大小设计，那么实际的图片肯定需要比 LsSize 的要大，大多少，太小了不够适应，太大了又浪费，如何取舍等问题，这一点取决的因素是什么，留给读者思考 ~~

一叶将在 GitHub 处建立一个[ScreenSolutions](https://github.com/leafsoar/ls-cocos2d-x/tree/master/ScreenSolutions) 项目，读者可以从这里参考实现的方案。(也许**此时**在 GitHub 所看到的实现并不完全，但已经有了简单的实现方法，并且能够运行，如有必要，将会新写一篇博客，去实现 ScreenSolutions 并且解说)
