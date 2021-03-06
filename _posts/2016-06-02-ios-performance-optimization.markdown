---
layout:     post
title:      "iOS 性能调优"
subtitle:   "ios-performance-optimization"
date:       2016-06-02
header-img: "img/bg15.jpg"
author:     "CMB"
tags:
    - iOS
    - 性能优化

---

影响 `APP` 的底层机制，主要是 `CPU` ， `GPU` 的占用率和内存的使用率，

`APP` 主线程在 `CPU` 中，计算显示内容，比如视图的创建，布局，图片解码，文件绘制等，计算完这些东西后交给 `GPU` 把结果提交到帧缓冲区去，等待下一次 垂直 信号到来时显示到屏幕上。由于垂直同步的机制，如果在一个 `VSync` 时间内，`CPU` 或者 `GPU` 没有完成内容提交，则那一帧就会被丢弃，等待下一次机会再显示，而这时显示屏会保留之前的内容不变。这就是界面卡顿的原因。

无论是 `CPU` 或 `GPU` 哪个阻碍了显示的流程都会造成掉帧的现象，在 `Xcode` 里面也可以看到手机 `APP` 在调试模式下 `CPU` 和内存的使用情况。

(1) `CPU` 方面

从三个打的方面考虑，对象的创建，调整和销毁都会影响 `CPU` 的使用

对象的创建如果不处理事件，尽量使用轻量级的 `CALayer` ，`xib` 或 `Storyboard` 会是这些创建的消耗大很多，经常使用的对象应该放到缓存里面重复利用，`UIImage imageNamed` 适合重复使用的对象， `initWithContentOfFile` 适合一次性使用的对象。

`CALayer` 并不能够直接调整视图的属性，而是通过 `runtime` ， `resolveInstanceMethod` 为对象临时添加一个方法，同时把对象的属性保存在一个字典里面，然后发送通知给代理，创建动画，`UIView` 是 `CALayer` 的包装，对对象的属性调整时大于一般对象的属性，应该有 `immutable` 的思想，另外重写 `drawreact` 方法也会带来很大性能的损耗，尽量避免视图层次过多的叠加和变动。`React` 有类似的思想，利用高效的算法，对比当前视图和之前视图的差异，然后做调整，当在移动开发时，

`React Native` 的效果和原生的还是有一定的差距，特别是动画的使用上，做频繁的变动导致掉帧率变大。导航视图移动过慢。

对象的销毁一般是在主线程里面来进行的，也可以单独的放到后台线程里面进行。对应大量使用循环对象，可以使用 `autoreleasePool` 来管理。

对应布局的技术和调整，可以考虑用空间换时间的方式，让服务器返回图片的大小，避免默认图的固定大小造成的 `tableview` 刷新过多，尽量缓存 `tableview` 的高度，重复使用，对应复杂的视图，可以考虑用串行队列，将对象的创建和push的动作隔离开来，可以明显的增加视图的加载效率，尽量使用后台计算视图的显示数据

`autolayout` 等技术使用在复杂布局上时，会产生严重的性能问题，应该避免使用，可以考虑 `ComponentKit` 、 `AsyncDisplayKit` ，`EZayout` 等框架，

对于文本的计算，可以使用 `coretext` 排版可以直接获取文本的高度，把计算高度的工作交给后台，`coretext` 对象可以重复使用。

对应图片的绘制，可以使用 `CoreGraphic` 获取 `bitmap` ，用 `ImageIO` 直接解压图片

(2) `GPU` 方面

`GPU` 能干的事情比较单一：接收提交的纹理（Texture）和顶点描述（三角形），应用变换（transform）、混合并渲染，然后输出到屏幕上。通常你所能看到的内容，主要也就是纹理（图片）和形状（三角模拟的矢量图形）两类。

超大图片的使用会导致内存的暴涨，可以使用 `GPU` 加速的 `layer` 控制。

申明透明的 `view opaque` 为1

`CALayer` 的 `border` 、圆角、阴影、遮罩（mask），`CASharpLayer` 的矢量图形显示，通常会触发离屏渲染（offscreen rendering），而离屏渲染通常发生在 `GPU` 中。当一个列表视图中出现大量圆角的 `CALayer`，并且快速滑动时，可以观察到 `GPU` 资源已经占满，而 `CPU` 资源消耗很少。这时界面仍然能正常滑动，但平均帧数会降到很低。为了避免这种情况，可以尝试开启 `CALayer.shouldRasterize` 属性，但这会把原本离屏渲染的操作转嫁到 `CPU` 上去。对于只需要圆角的某些场合，也可以用一张已经绘制好的圆角图片覆盖到原本视图上面来模拟相同的视觉效果。最彻底的解决办法，就是把需要显示的图形在后台线程绘制为图片，避免使用圆角、阴影、遮罩等属性。可以考虑用画图的方式截取圆角。

比较高效的开源库有 `AsyncDisplayKit` ，拥有高效的异步绘制和渲染能力，最后，用 `Instuments` 的 `GPU Driver` 预设，能够实时查看到 `CPU` 和 `GPU` 的资源消耗。在这个预设内，你能查看到几乎所有与显示有关的数据，比如 `Texture` 数量、`CA` 提交的频率、`GPU` 消耗等，在定位界面卡顿的问题时，这是最好的工具。如果界面出现不明卡顿，可以监听主线程的 `runloop`，监控到了卡顿现场, `PLCrashReporter` ,它不仅可以收集 `Crash` 信息也可用于实时获取各线程的调用堆栈。