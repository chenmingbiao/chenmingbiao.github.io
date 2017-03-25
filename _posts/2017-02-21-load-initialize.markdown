---
layout:     post
title:      "load 和 initialize"
subtitle:   "load-initialize"
date:       2017-02-21
header-img: "img/bg18.jpg"
author:     "CMB"
tags:
    - iOS
    - 初始化

---

## load

`load` 会在类或分类被添加到 `runtime` 时调用。并且只会被 `runtime` 调用一次。

如果子类没有实现，父类的 `load` 方法也不会被再次调用。

### load 的调用顺序

* 链接的 `framework`
* 自己的 `image`
* `C++` 静态初始化方法，具有 `__attribute__(constructor)` 修饰的函数
* 链接到你的 `image`

其它：

* 父类优先子类
* 类优先分类（ `category` ）

## initialize

`initialize` 会在类被使用前调用。包括类方法。

比如下面的代码。在 `load` 内调用 `[self class]` 会导致 `initialize` 被调用。

```objc
+ (void)load
{
    NSLog(@"%@ %s", [self class], __FUNCTION__);
}

+ (void)initialize
{
    NSLog(@"%@ %s", [self class], __FUNCTION__);
}
```

每个类都会被 `runtime` 线程安全的调用一次 `initialize` 方法。

父类会在子类前被调用。

如果子类没有实现或者调用 `[super initialize]` ，则父类会被再次调用。

下面的代码可以保证只执行一次初始化操作：

```objc
+ (void)initialize {
	if (self == [ClassName self]) {
	  // ... do the initialization ...
	}
}
```

### 参考资料

1、[Objective-C +load vs +initialize](http://blog.leichunfeng.com/blog/2015/05/02/objective-c-plus-load-vs-plus-initialize/)

2、[NSObject +load and +initialize - What do they do?](http://stackoverflow.com/questions/13326435/nsobject-load-and-initialize-what-do-they-do)