---
layout: post
title: Cocos2d-x 屏幕适配新解 - 兼容与扩展
date: 2013-05-13 8:02
comments: true
categories: Cocos2d-x
---

在读这篇文章之前，先读前一篇文章 [Cocos2d-x 屏幕适配新解](http://blog.leafsoar.com/archives/2013/05-10-19.html) 是必要的。

如果说前一篇文章文章在 LsSize 提出之前的是基础，LsSize 是应用，那么对于这篇文章来说，LsSize 是基础，而这里是其的综合应用，我之初衷是其扩展性和兼容性，激发读者思维。也许你并没有体会出 LsSize 的强大，而实际上，**它能做的比你想象的要多的多，这是前话 ~**

## ShoAll 模式的兼容
首先 LsSize 能满足 ShowAll 模式的需要，因为开始就是把 LsSize 当作 ShowAll 中的 WinSize 来设计的。 并且可以为背景做装饰，而在游戏设计之时并没有什么区别，LsSize 可以设定显示区域的大小，使背景层与 LsSize 分离（这一点在上一篇文章最后已经提到），从而保证了游戏的元素不会超出 LsSize 而露出到 VisibleSize 的区域内。

<!-- more -->

## NoBorder 模式兼容
为什么说 NoBorder **兼容**模式，它本身不就是 NoBorder 么，它与实际的 NoBorder 区别又在何处，有何优势？首先说说兼容，使用此模式，**并不影响** 你继续使用 VisibleSize 和 VisibleOrigin(以后简写 **Visible**)，你可以不使用 LsSize 的参考点，而使用 **Visible** 的相关值获取屏幕的拐点，游戏元素按照 Visible 来设置也可。下面详细介绍基于 LsSize 的 NoBorder 和原油 NoBorder 的区别以及其优势。

我们设想这样一个 **实际情况** 。我们需要一套资源图片，做为在适合分辨率的资源展示，当屏幕的大小分辨率在 **854x480，800x480，728x480** (横屏下：为什么高度同样是 480 而宽度有这么多值 : P 这也是屏幕适配万恶之一了吧) 时，我们使用一套资源，当高度小于 480 时，我们使用另一套小的资源是合理的设计。而这里我们的资源宽姑且先不论，高一定是 480 最为合适了，最接近此分辨率的图片。那我们使用 NoBorder 的时候该 **设置 WinSize 为多少** 了呢？

<s>基于 854x480 设计！好，那么当程序跑在 854x480 的屏幕上，正好满屏显示，而图片资源并没有放大或者缩小，或者说基于像素点点对点显示的。但是当这样设计的 WinSize 跑在 800x480 和 728x480 分辨率会如何？也许已经知道了，为了保证小于 854 那一小块区域的显示，画面将会缩小那么一点点，也许在如今屏幕的 ppi 日益渐高的情况下，并不十分明显，但画面一定是有那么一点模糊了。同理可以遇见，**不论 WinSize 如何设置，在 三种少许不同分辨率下，显示的效果肯定略有不同。** 而分辨率差别越大，这种效果就越明显。</s>

**注：细心的朋友已经读出上文描述中出现的 Bug ，并多谢指出问题的朋友。下面修复 Bug 并重新描述问题的情况 ～**

折衷方案，我们基于 800x480 设计，那么此时出现的情况是，当跑在 800x480 的屏幕上时，正好满屏显示，而图片并没有任何放大或者缩小，或者说基于像素点点对点显示的。而当这样设计的 WinSize 跑在 728x480 和 854x480 的分辨率会如何？ 854x480 相比 800x480，**前者的宽高比要大于后者，所以它是宽对齐**的，这意味着，画面有所放大，而上下将会有一部分残缺，**此时设计高度将会失去参考价值**。728x480 相比 800x480 ，**前者的宽高比小于后者，所以它是高对齐**的，此时画面并没有缩放，只是横向截取了小部分，这样的情况是由于 NoBorder 的实现机制所决定。**当然我们可以将设计的宽度设置的很宽很宽，以保证高对齐，哈～但是魔(设计宽度)高一尺，道(实际宽度)高一丈,倒不如使用后文提到的“固定高度”方式了~** 而这里的 854x480，800x480，728x480 等数据只是屏幕适配等问题的**“缩影”**。

读到这里也许已经发现了，LsSize 已经完美 **(**这里的完美，并非只此特殊情况下的解决方法，而是总览全文，基于 LsSize  的设计理念，其兼容性和可扩展性，显然在此时，固定高度是更好的实现方式**)** 的解决了这个问题，动态 WinSize ，一个合理的设计，我梦将 LsSize 设定为 720x480，并且使用 高度为 480 的图片资源，而宽度可以往大了设计，比如 854x480  的图片资源，读过前文 LsSize 的实现原理，我们可以知道，**在这三种情况下，屏幕的画面并没有缩放，因为实际的宽度总是大于 720 ，从而达不到缩放的条件。**480 高度的图片，能够正好填充 480 高度的屏幕，而图片的宽度往大了设计，在宽度稍微小的屏幕下，会被截取一部分，但就显示效果来说，并没有什么损失，而游戏的元素位置可以用原来的方法基于 Visible 来设计。

为什么可以做到如此！原本的  NoBorder 通过固定 WinSize 根据屏幕宽高比缩放，所以会有不同程度的缩放，而 LsSizeNoBorder 的设计实现，通过屏幕宽高比来获得 WinSize 的值，以保证 LsSize 总能在屏幕上正好全部显示。

LsSizeNoBorder 比 NoBorder 好，好多少，这就仁者见仁，智者见智了 ~

## kResolutionFixedHeight，kResolutionFixedWidth 扩展兼容模式
FixedHeight 和 FixedWidth 是什么模式，如果你试用了最新版的  cocos2d-x (2.1.3)就能发现这两种模式，一种是固定设计时的高，一种是固定设计时的宽。而在当前的 2.0.4 并没有这两种模式，而现在，你可以通过 LsSize 来实现这两种模式。存在既是合理，FixedHeight 和 FixedWidth 的存在是合理的，比如我们写一个横版过关游戏，同样是三种分辨率 **854x480，800x480，728x480** 使用固定设计高度的方法，可以避免在 NoBorder 中会根据宽度而做的缩放。

而在 LsSizeNoBorder 中好似也实现了相同的功能！但注意有一点区别，在 LsSizeNoBorder 中，实际屏幕高度为 480 ，如果宽度小于 720 时，那么画面会缩放，而这里新模式固定高度不会。如果我们想让 LsSize 实现这种功能怎么做，我们将对 LsSize 的设计稍作扩展，上篇文章的代码：

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

我们的实际缩放系数，是根据 scaleX 和 scaleY 的大小来判断，依据哪个方向缩放，从而在显示效果上是高对齐还是宽对齐。而想要固定是高度对其还是宽度对其，那只要换用如下方法即可：

``` c++
	// 要实现这种功能，我们需要做的就是算得 缩放系数，缩放系数由 原来的设计稍作演变即可
	// 由于 NoBorder 的缩放是根据 scaleX 和 scaleY 的熟大熟小来判断缩放系数是参照横向还是竖向
	// 固我们需要两个先决条件，固定的方向 和 缩放的参照方向，而得到如下算法
	
	// 固定高度
	if (scaleX > scaleY)
		scale = scaleX / (frameSize.height / (float) lsSize.height);
	else
		scale = scaleX / (frameSize.width / (float) lsSize.width);

	// 固定宽度
	if (scaleX > scaleY)
		scale = scaleY / (frameSize.height / (float) lsSize.height);
	else
		scale = scaleY / (frameSize.width / (float) lsSize.width);
```		

	
下面通过几张效果图展示 **固定高度** 和 **固定宽度** 效果：

显示效果：（NoBorder **固定高度**模式 ;FrameSize = 520x320; LsSize = 480x320; WinSize = 动态获取）
![图片](/images/2013/screen-resolution-extension-1.jpg)

显示效果：（NoBorder **固定高度**模式 ;FrameSize = 520x360; LsSize = 480x320; WinSize = 动态获取）
![图片](/images/2013/screen-resolution-extension-2.jpg)

显示效果：（NoBorder **固定宽度**模式 ;FrameSize = 480x360; LsSize = 480x320; WinSize = 动态获取）
![图片](/images/2013/screen-resolution-extension-3.jpg)

显示效果：（NoBorder **固定宽度**模式 ;FrameSize = 480x300; LsSize = 480x320; WinSize = 动态获取）
![图片](/images/2013/screen-resolution-extension-4.jpg)

如图所示，我们固定了一个方向，使得这个方向上的设计长度正好填充屏幕，而另一个方向上会有所延伸或截取，而此时如果想或者屏幕拐点，可以配合 Visible 的显示区域算得。而这也正式 cocos2d-x 2.1.3 所实现的功能，而如果你此时为了稳定而使用 2.0.4 stable 版本，那么就**可以通过这种基于 LsSize 的设计方法实现 FixedHeight 与 FixedWidth。** 而在将来后续版本稳定，也可以很平滑的升级到使用自带的方式替换，其显示效果一样，只是后续版本 cocos2d-x 在内部将它封装了而已。

## kResolutionLeafsoar 模式的核心思想

透过现象看本质！基于固定 LsSize 的动态 WinSize 设计。之所以能够兼容这么多模式并且有所加强，在于 LsSize 在 FrameSize、WinSize、VisibleSize、VisibleOrigin等概念之外的存在，并且通过动态计算 scale  而游走于此等之间。它的存在并不依赖于 这些已有概念，而反过来，让已有的概念去依赖 LsSize 。从而保持设计上的灵活性与扩展性。
