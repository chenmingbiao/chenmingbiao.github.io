---
layout:     post
title:      "Swift 中类的两段式构造"
subtitle:   "swift-two-phase-initialization"
date:       2017-01-20
header-img: "img/bg17.jpeg"
author:     "CMB"
tags:
    - iOS
    - Swift
    - 基础

---

### 两段式构造

#### 第一阶段：

1. 程序调用子类的某个构造器
2. 为实例分配内存, 此时实例的内存还没有被初始化
3. 指定构造器确保子类定义的所有实例存储属性都已被赋初值
4. 指定构造器将调用父类的构造器, 完成父类定义的实例存储属性的初始化
5. 沿着调用父类构造器的构造器链一直往上执行, 直到到达构造器链的最顶部

#### 第二阶段：

1. 沿着继承树往下, 构造器此时可以修改实例属性和访问self, 甚至可以调用实例方法
2. 最后, 构造器链中的便利构造器都有机会定制实例和使用self

		为了使得构造过程更加安全, Swift进行了安全检查

### 安全检查

* 安全检查 1：指定构造器必须先初始化当前类中定义的实例存储属性, 然后才能向上调用父类构造器
* 安全检查 2：指定构造器必须向上调用父类构造器, 然后才能对继承得到的属性赋值
* 安全检查 3：便利构造器必须先调用同一个类的其他构造器, 然后才能对属性赋值
* 安全检查 4：构造器在第一阶段完成之前, 不能调用实例方法, 不能读取实例属性

```swift
class Person {
     
    var height: Double
    var weight: Double
     
    // 定义指定构造器
    init(height: Double, weight: Double) {
        self.height = height
        self.weight = weight
    }
     
    // 定义便利构造器(使用convenience修饰)
    convenience init(h height: Double, w weight: Double) {
        self.init(height: height, weight: weight)
    }
}
 
class Man: Person {
     
    var sex: String!
     
    init(sex: String, height: Double, weight: Double) {
        // print(super.name) 不能再父类初始化之前调用父类中的属性
        super.init(height: height, weight: weight)
        super.name = "cmb"
        print(self.name)
        // print(self.sex) 不能在本类中的属性没有进行初始化的时候进行调用
        // 会出现:fatal error: unexpectedly found nil while unwrapping an Optional value错误
        self.sex = sex
        print(self.sex)
    }
     
    convenience init(s sex: String, h height: Double, weight: Double) {
         
        // 在调用其他构造器之前, 不能访问或修改任何实例存储属性
        // print(self.name) 错误
        // super.name = name 错误
        self.init(sex: sex, height: height, weight: weight)
    }
}
 
var man = Man(sex: "男", height: 178, weight: 62.0)
```