---
layout:     post
title:      "iOS 中静态链接库的使用"
subtitle:   "how-to-use-static-library-in-ios"
date:       2016-07-31
header-img: "img/bg16.jpg"
author:     "CMB"
tags:
    - iOS
    - 编译库

---

### 什么是编译库

与 Java , .net 一样，objc 中也是有编译库的概念，主要用于 SDK (如百度地图SDK)，库是编译后的二进制文件，主要是开发一些接口给开发者使用，而这种接口的实现却不能暴露的场景。

### 编译库的分类

#### 静态库：

 1. `.a` 文件：纯二进制文件，需要配合 `.h` 文件一起使用，通过 `.h` 文件引用库里面的内容

 2. `.framework` 文件：包含二进制文件

> 链接时，静态库会被完整地复制到可执行文件中，例如iOS程序打包的时候会包含所有引用的静态库

#### 动态库： 

 1. .dylib文件: 和静态库一样也是纯二进制文件

 2. `.framework` 文件：和静态库的 `framework` 一样，只是它由 .dylib 组成

> Xcode7以后iOS7以及以下的iOS不可以使用三方动态库

### 如何生成和使用静态链接库

#### 新建项目，选择 `Framework & Library` , 再选择 `Cocoa Touch Static Library` ，点确定。

![](/img/how-to-use-static-library-in-ios-1.png)

#### 我建一个叫 `BCTestLib` 的项目，然后编写我们的代码

`BCTestLib.h` :

```
@interface BCTestLib : NSObject

+ (void)helloWorld;

@end
```

实现如下：

`BCTestLib.m` :

```
@implementation BCTestLib

+ (void)helloWorld{
    NSLog(@"hello world!");
}

@end
```

#### 然后创建一个 `Category`（等下你就知道为什么我要创建`Category`），我创建了一个叫 `BCSayBye` 的分类，代码如下：

`BCTestLib+BCSayBye.h` :

```
@interface BCTestLib (BCSayBye)

+ (void)sayBye;

@end
```

实现如下：

`BCTestLib+BCSayBye.m` :

```
@implementation BCTestLib (BCSayBye)

+ (void)sayBye{
    NSLog(@"say bye!");
}

@end
```

#### 添加自己的类和 `category`

编译的时候需要将头文件拷贝到生成的库路径下，这里的头文件是用于给外部使用的，一般是把库里面的文件放在一个头文件中引用，这样外部在使用的时候直接引用该头文件即可

![](/img/how-to-use-static-library-in-ios-2.png)

> 在 `Copy Files` 默认是不会添加 `category` 的头文件，所以需要我们手动去操作

#### 设置支持的最低版本和最高版本

(1) `Base SDK`：是当前类库是基于哪个版本的SDK开发的，也就是最高支持的SDK

(2) `Deployment Target`：类库支持的最低版本

![](/img/how-to-use-static-library-in-ios-3.png)

![](/img/how-to-use-static-library-in-ios-4.png)

#### 配置编译选项

由于我们编译的是类库，在使用的时候需要支持Debug和Release两种模式下，需要编译所有的 `architecture` 版本
　　
![](/img/how-to-use-static-library-in-ios-5.png)

#### 编译（Cmd + B）

我们分别切换到模拟器和真机模式进行编译，在真机模式下编译完成后， `Products` 中的文件会变正常（原来为红色）　　　

> 注意，需要设置Build Release 版本

![](/img/how-to-use-static-library-in-ios-6.png)

![](/img/how-to-use-static-library-in-ios-7.png)

![](/img/how-to-use-static-library-in-ios-8.png)

#### 编译完成

`libBCTestLib.a` 右键，`Show in Finder` 就能看到我们编译好的静态库

![](/img/how-to-use-static-library-in-ios-9.png)

#### 合并 `.a` 文件(可跳过)

上面看到，编译后的用于模拟器的静态库和用于真机的静态库不一样，每次切换的适合都得重新引用 `.a` 文件，这样显得特别麻烦，苹果提供了一个合并多个 `.a` 文件的方法，合并后的 `.a` 文件真机和模拟器都支持（合并后大小为原来两个文件大小之和）

在终端通过命令合并

`lipo –create Release-iphoneos/libBCTestLib.a Release-iphonesimulator/libBCTestLib.a –output libBCTestLib.a`

还有一种方法可以动态的引用静态库，就是通过配置工程的库引用路径和编译标示，编译的适合 `Xcode` 会根据当前的环境自动找到相关的 `.a` 库。

#### 使用

我们创建一个iOS项目，吧相关的 `.a` 文件和 `.h` 文件拖到我们的项目中，拖入后，`Xcode` 会自动把静态库添加到工程

![](/img/how-to-use-static-library-in-ios-10.png)

***注意***

(1) 头文件也要引入到工程里面（不然用不了）

(2) 模拟器和真机对应的 `.a` 文件不一样，根据需要引用 `.a` 文件

(3) 如果静态库内有 `category` 分类，那么需要在添加 `-ObjC` 编译标识，否则可能会报：`unrecognized selector sent to instance`

![](/img/how-to-use-static-library-in-ios-11.png)

***其他编译参数说明***　　　　　　　　

(1) `－ObjC`：加了这个参数后，链接器就会把静态库中所有的Objective-C类和分类都加载到最后的可执行文件中。

(2) `－all_load`：会让链接器把所有找到的目标文件都加载到可执行文件中，但是千万不要随便使用这个参数！假如你使用了不止一个静态库文件，然后又使用了这个参数，那么你很有可能会遇到ld: duplicate symbol错误，因为不同的库文件里面可能会有相同的目标文件，所以建议在遇到-ObjC失效的情况下使用-force_load参数。

(3) `-force_load`：所做的事情跟-all_load其实是一样的，但是-force_load需要指定要进行全部加载的库文件的路径，这样的话，你就只是完全加载了一个库文件，不影响其余库文件的按需加载。

引用自 [关于Xcode上的Other linker flags](http://www.cnblogs.com/robinkey/archive/2013/05/27/3101095.html)

(4) 如果静态库中采用 `ObjectC++` 实现，或者静态库使用 `C/C++` 写的，在调用的时候可能出错，因此需要您保证您工程中至少有一个 `.mm` 后缀的源文件(您可以将任意一个 `.m` 后缀的文件改名为 `.mm` )

或者在工程属性中指定编译方式，即将 `XCode` 的 `Project -> Edit Active Target -> Build -> GCC4.2 - Language -> Compile Sources As` 设置为 `Objective-C++`

引用自 [xcode中引入静态库文件方法](http://blog.csdn.net/zhangkongzhongyun/article/details/8047500)

#### 运行

代码如下：

```
#import "ViewController.h"
#import "BCTestLib+BCSayBye.h"

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [BCTestLib helloWorld];
    [BCTestLib sayBye];
}

@end
```

运行效果：

![](/img/how-to-use-static-library-in-ios-12.png)

### 参考资料

1、[iOS 开发中 动态库 与静态库的区别](http://www.tuicool.com/articles/VFFjmq6)

2、[Mac OS X上的lipo命令详解](http://blog.chinaunix.net/uid-24512513-id-3385418.html)