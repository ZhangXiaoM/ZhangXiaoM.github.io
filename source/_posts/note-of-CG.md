---
title: note of CG
date: 2018-07-19 18:00:35
tags: 动画
---

------

#### Q： [The relationship between CoreGraphics, UIViews and CALayers](https://stackoverflow.com/questions/17715530/the-relationship-between-coregraphics-uiviews-and-calayers)

 我经常使用 `CoreGraphics` 和 `CoreAnimation`，我知道它们是如何独立工作的，我想知道它们的一些交集，我也知道 `UIView` 是 `CALayer` 的封装，`CALayer` 负责沉重的渲染工作，`UIView` 负责响应事件。

但是，我所知道的都是它们如何独立工作，我想知道它们之间如何协同工作，特别是 `CoreGraphics` 和 `CALayer`。

`CoreGraphics` 和 `CALayer` 的关系是什么？

我的理解是，`CALayer` 封装 `CoreGraphics` 的方法来绘制它自己，但是只绘制一次，并且持有绘制的快照，直到失效，但是，这些绘制方法怎么使子 `layer` 和父 `layer` 相互作用的呢？

例如，当我绘制拥有子视图的UIView时会发生什么，并且我重载了drawRect方法？这对它的子图层的绘制有何影响？

 将两者混合在同一个函数中是不是一个好主意？

1、[confusion regarding quartz2d, core graphics, core animation, core images](https://stackoverflow.com/questions/9248530/confusion-regarding-quartz2d-core-graphics-core-animation-core-images)。

这个问题询问了彼此之间的差异，并且所选择的答案确实提供了，但它为每个单独的库提供了答案，就像另一个不存在一样。

 2、[To Drawrect or not to Drawrect](https://stackoverflow.com/questions/14659563/to-drawrect-or-not-to-drawrect-when-should-one-use-drawrect-core-graphics-vs-su?rq=1)

另一个很好的问题，但它只涉及绘制CoreGraphics与将问题交给UIKit的主题，但无论如何，选择的答案提供了部分难题。

 3、[Animating Pie Slices with Custom CALayer](http://blog.pixelingene.com/2012/02/animating-pie-slices-using-a-custom-calayer/).

必须是我在这个主题中看到的最有价值的教程之一，它是唯一一个指导我绘制CALayer的教程

4、[What is different between CoreGraphics and CoreAnimation](https://stackoverflow.com/questions/4392918/what-is-different-between-coregraphics-and-coreanimation)

 

#### A：

也许考虑它的最简单的方法是：在GPU的内存中，你有纹理，这些纹理是位图图像。CALayer可能具有与之关联的纹理，或者可能没有。这些情况分别对应于具有-drawRect：方法的图层和仅包含子图层的图层。从概念上讲，每个具有与之关联的纹理的图层都具有不同的纹理（有一些细节和优化使得这不是严格/普遍正确的，但在一般的抽象情况下，它可以帮助将其想象一下办法）。考虑到这一点，父图层的-drawRect：方法对其任何子层的-drawRect：方法没有影响，并且（在一般情况下）子层的-drawRect：方法对其父层的-drawRect：方法没有影响。每个 layer 都绘制自己的纹理（也称为“后备存储”），然后，基于图层树和相关的几何和变换，GPU将所有这些纹理组合成您在屏幕上看到的内容。当其中一个层直接或间接无效时（通过`-setNeedsDisplayInRect:`)，然后，当CA开始在屏幕上显示下一帧时，将通过调用其`-drawRect:`方法重绘无效层。这将更新相关的纹理，一旦更新了所有无效图层的纹理，GPU将合成它们，生成您在屏幕上看到的最终位图。

所以回答你的第一个问题：在一般情况下，不同 layer 的 `drawRect:` 方法之间没有相互作用。

第二个问题：对于图层的绘制和渲染来说，你可以将 `UIView` 和 `CALayer` 看成一个东西。

第三个问题：有什么好主意能将 `UIView` 和 `CALayer` 混合使用吗？每一个 `UIView` 都有一个 `CALayer` ，所以在某种程度上他们已经混合工作了，向 `UIView` 的 layer 上添加子图层是没有任何问题的，但是子图层不能得到任何 `UIView` 的功能，比如添加手势，如果该图层的目的只是为了生成要显示的图像，那就没问题。如果你希望子视图响应触摸手势或者使用 `AutoLayout` 布局，请使用 `UIView`。最好将UIView视为CALayer，并添加一些额外的功能。

CoreGraphics是一个用于绘制2D图形的API。CG（但不是唯一的）的一个常见用途是绘制位图图像。CoreAnimation是一个API，提供了一个有组织的框架，用于在屏幕上操作位图。你可以在没有调用CoreGraphics绘图原语的情况下有意义地使用CoreAnimation。

CoreGraphics将2D渲染转换为位图，然后可以通过CALayer内容属性移动到图形卡。正确实施，CoreGraphics解决方案可以像直接OpenGL方法一样快，因为CoreGraphics和CALayer无论如何都是建立在OpenGL之上的。



-------

### Q：[to drawRect or not to drawRect (when should one use drawRect/Core Graphics vs subviews/images and why?)](https://stackoverflow.com/questions/14659563/to-drawrect-or-not-to-drawrect-when-should-one-use-drawrect-core-graphics-vs-su)

我知道如何使用子视图和使用drawRect创建复杂的视图。我试图完全理解何时使用一个而不是另一个。

我的很多困惑来自学习如何使表格视图滚动性能真正流畅和快速。基本上它说要使表滚动黄油光滑，秘诀是不使用子视图，而是在一个自定义uiview中完成所有绘图。从本质上讲，使用大量子视图似乎会减慢渲染速度，因为它们有很多开销，并且不断在父视图上重新合成。

公平地说，这是在3GS是一个非常新的品牌时写的，iDevices从那以后变得更快。当我得到一个主意说，要尽可能的使用 CG 时，我又被告知 -drawRect 会导致 APP 性能变差。

我看到一些使用Core Graphics的情况：

1、你有动态数据（Apple的股票图表示例）。

2、您有一个灵活的UI元素，无法使用简单的可调整大小的图像执行。

3、您正在创建动态图形，一旦渲染，将在多个地方使用。

我看到避免Core Graphics的情况：

1、视图的属性需要单独设置动画。

2、你有一个相对较小的视图层次结构，所以使用 CG 带来的好处不太明显。

3、您想要更新视图的某些部分而不重绘整个视图。

4、父视图大小更改时，子视图的布局需要更新。

您在什么情况下可以使用drawRect / Core Graphics（也可以通过子视图完成）？如何/为什么建议将内容全部绘制在表格视图单元格的同一层级，Apple 为什么认为 -drawRect 影响性能？那么简单的背景图片（你何时使用CG创建它们与使用可调整大小的png图像）？

1、如果你用 CG 绘制 和 用 `UIImageView` 以及一个预渲染的 png 图片得到一样的效果，选择哪个？

2、什么情况下用 CG 绘制？（可能在元素的显示是可变的时候。例如一个20种不同颜色变化的按钮。还有其他情况吗？）

3、通过在复杂的UIView渲染自身后有效捕获单元格的快照位图，和在滚动和隐藏复杂视图时显示，可以获得表单元格相同的性能提升吗？

 

 

 

 

 

 

 

 

 

 

 

 

 

 



