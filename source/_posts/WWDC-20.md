---
title: WWDC 20 的那些事儿
date: 2020-08-02 16:35:02
tags: [iOS, WWDC]
---

# 前言

　　WWDC 20 也过去一段时间了，这次的 WWDC 带给了开发者们不小的惊喜，所以我在这里也尝试记录一下 WWDC 20 之后的新变化。

<!-- more -->

## Swift

　　众所周知，`Swift`是苹果现在和未来主推的一门语言。国内外有很多的  APP 项目已经从`Objective-C`迁移到了`Swift`。这次的更新想必也会想让更多的开发者蠢蠢欲动转为``Swift Coder`。



### Runtime Performance



#### Code Size

　　苹果一直在观察`Swift`项目产物中的`__text`的大小，从下图我们可以看出，在`Swift 5.3`里，`Code Size`已经变成了`OC`的 1.5 倍。

![SwiftCodeSize](/img/WWDC20/SwiftCodeSize.png)

　　我们之前常说`OC`是一门很啰嗦的语言，这一点可以在方法名的调用里看出来，一个多参数的方法调用会生成很长的代码，这样一点也不美观，而生成的`Code Size`也相应的很大。而`Swift`在这方面表现的好很多，但为什么`Code Size`还越来越多了呢？官方的解释是因为`Swift`带有一些安全相关的 Feature，这些在生成的产物中需要有一定量的代码实现，从而增加了 Size。



#### Dirty Memory

　　有`Dirty Memory`，相应的就有`Clean Memory`。对于`iOS`来说`Clean Memory`指的就是可以被重新创建的内存，主要分为以下几类。

* App 的二进制文件
* framework 中的 _DATA_Const 段
* 文件映射的内存
* 未写入数据的内存



　　相应的，`Dirty Memory`指的就是不能被系统回收的内存占用。

　　现在我们有这么一段代码

```objective-c
// Model
@interface Mountain: NSObject
{
    NSUUID *uuid;
    NSString *name;
    float *height;
}
@end
    
NSArray<Mountain *> *mountains = loadMountains()
```

　　众所周知，`OC`中的`NSArray`存的是对应元素的指针，这个从深浅拷贝就可以看出来。由于`Tagged Pointer`，某些`NSString`比较短（没有超过 8 字节），无需用指针存储对象以达到优化。

　　在`Swift`中，`Array`的直接存储值（`Swift`中数组是直接存储值）最大上限变成了 16 个字节， 15 位的长度可以应对更多的场景。

　　于是`Mountain`在两种语言中会有这种区别：

![dirty memroy](/img/WWDC20/SwiftArray.png)

#### Swift 标准库下沉

　　`Swift Standard Library`现在变成了 `Foundation`的依赖。后续苹果应该会完善 API ，那些只能通过`C`和`OC`访问的方法也会逐步迁移。



### 开发



#### 诊断策略

　　新版本的`Swift Compiler`中，更新了新的诊断策略方案

* 使用全新定位代码问题的策略
* 更加精确提示以及可行性更高的提示方案



#### 代码补全

　　从以下的例子可以看出，`Swift`的代码提示有了更进一步的加强

![三元表达式的补全](/img/WWDC20/三元表达式补全.png)

![KeyPath](/img/WWDC20/KeyPath.png)

　　Xcode 12 的代码补全速度也有了较大的提高。



### 语法

#### 多尾随闭包

　　在之前的`Swift`中，尾随闭包只允许当成最后**一个**参数，现在可以有多个尾随闭包了。

```swift
UIView.animate(withDuration: 0.3) {
    self.view.alpha = 0
} completion: { _ in 
    self.view.removeFromSuperview()
}
```

#### KeyPath 函数化

　　在`Swift 4.1`中引入了`KeyPath`。新版本中我们可以把`KeyPath`作为函数使用。

　　下面的例子里，我们使用`KeyPath`访问每个属性的`shoeSize`属性代替复杂的闭包。

![函数化](/img/WWDC20/KeyPathMethod.png)

#### 隐式 self

　　在以前的`Swift`版本中，我们必须显式声明`self`，而新版本则在闭包中增加隐式`self`，这个在`SwiftUI`中很有用处。

　　但是对于循环引用的闭包，看来`[weak self] 和 guard let self = self else {return}`还是无法避免显示声明。

![隐式 self in SwiftUI](/img/WWDC20/Self.png)

#### 多模式捕获

　　之前的`Swift`中，我们用`try Catch{}`往往要进行一些类型判断或者`switch case`。

　　在 5.3 中，我们可以使用多个 `catch`来达到`switch`的效果。

![多模式捕获](/img/WWDC20/Switch.png)

#### 枚举

* 比较

　　当`enum`遵循`Comparable`时就可以对枚举进行比较。

![比较](/img/WWDC20/enumCompare.png)

* enum cases as protocol witnesses

　　当我们的枚举遵循一些协议的时候，我们不再需要重复写一遍方法名。

![enum cases as protocol witnesses](/img/WWDC20/enum cases as protocol witnesses.png)

# 后记

　　笔者已经在早些时候开始转向`Swift`并在极力 push 公司的项目向`Swift`转化。

　　看了这次的`WWDC 20`，这样的`Swift`，怎能不爱？

# 参考

[What's new in Swift](https://xiaozhuanlan.com/topic/2804537169)

