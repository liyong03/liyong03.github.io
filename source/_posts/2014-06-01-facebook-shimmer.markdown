---
layout: post
title: "Facebook shimmer 实现原理"
date: 2014-06-01 14:33:57 +0800
comments: true
categories: ios programming
---

Facebook 最新的 App [Paper](https://itunes.apple.com/us/app/paper-stories-from-facebook/id794163692?mt=8) 包含了很多丰富的动画元素，为此 Facebook 甚至设计了一款 iOS 的动画引擎 [pop](https://github.com/facebook/pop)。整个 Paper 的动画设计中另一个很有特色的动画是文字的闪光效果。

![Shimmer](/images/shimmer.gif)

这个效果其实和 iPhone 的锁屏文字效果非常类似。

![lock screen text animation](/images/ios_lock_text.gif)

#####Facebook Shimmer
Facebook 最近在 Paper 中实现这种文字效果的部分开源了出来，就叫做 [Shimmer](https://github.com/facebook/shimmer)。使用 Shimmer 可以很容易实现这种文字闪光的效果。

但是 Shimmer 不仅仅局限于对文字做删光效果，它可以实现任何 view 的闪光效果，如果使用图片做内容，可以实现图像内容的闪光。这样可以实现某些 logo 的闪光效果，甚至可以用来实现反光效果。

#####Shimmer 原理分析
Shimmer 是怎么实现这种效果的呢？通过阅读源码可以了解到（Shimmer 代码非常短，很容易阅读），其内部是依靠 Core Animation 来实现的。

Shimmer 的内部包含两个 CALayer， 一个是展示内容的 contentlayer， 另一个是完成闪光效果的 masklayer。

这两个 CALayer 通过两个步骤来实现闪光: 遮罩（mask） 和 滑动（slide）。

###### 1: 遮罩（Mask）
顾名思义，遮罩的作用就是要让闪光效果只是对文字部分起作用，而不是文字所在的整个 view。这一步通过 CALayer 的 mask 属性实现。Shimmer 将 masklayer 设置为 contentlayer 的 mask。先来看看 CALayer 关于 mask 的解释：
    
>The layer’s alpha channel determines how much of the layer’s content and background shows through. Fully or partially opaque pixels allow the underlying content to show through but fully transparent pixels block that content.

>The default value of this property is nil nil. When configuring a mask, remember to set the size and position of the mask layer to ensure it is aligned properly with the layer it masks.

Shimmer 中利用这一点，通过设置 masklayer 中 alpha 通道的效果来实现闪光效果中的高亮部分。首先将 contentlayer 中的内容设置为白色，通过 masklayer 的遮罩，对应 masklayer 中 alpha 小的部分将会透出一些 contentlayer 的背景，而 alpha 大的部分将会显示出白色的高亮。剩下的工作就是要让这些高亮动起来了。

###### 2: 滑动
要实现滑动效果，其实非常简单，我们可以将通过设置 masklayer 的 position 来控制其与 contentlayer 做 mask 的部分，那么，我们可以通过将 masklayer 的尺寸放大，然后保证调整 position 的时候能完全遮罩住 contentlayer 即可，这样就实现了闪光部分的移动，从而实现了闪光效果。下面这张图很好的描述了整个滑动的原理（图片取自[这里](http://css-tricks.com/slide-to-unlock/)）：

![](/images/animatedbackground.png)

通过调整 masklayer 中 alpha 通道的形状，可以实现不同的闪光效果。例如如果使用圆形光斑效果的 alpha 通道，那么就可以实现 iPhone 本身锁屏的文字滑动效果了。

#####总结
从整个 Shimmer 的实现来看，Core Animation 本身强大的功能确实为 iOS 实现很多漂亮的动画带来了可能。但是大部分 iOS 开发者却没有深度掌握和挖掘出这些潜能来。Facebook Shimmer 对于我们或许能够带来某种启发，iOS 系统中很多看似复杂的动画效果其实都可以用 Core Animation 来实现，而不是某些不存在的私有 API。

另 1：对于想要更深入了解 Core Animation 的可以参考阅读 [Objc.io 第12期](http://www.objc.io/issue-12/)和[《iOS Core Animation Advanced Techniques》](http://www.amazon.com/iOS-Core-Animation-Advanced-Techniques-ebook/dp/B00EHJCORC/ref=sr_1_1?ie=UTF8&qid=1401552094&sr=8-1&keywords=ios+core+animation+advanced+techniques)。

另 2：[这里](http://css-tricks.com/slide-to-unlock/)是一篇利用 css 实现 iPhone 锁屏文字动画的文章，原理与本文描述类似，可以参考。
