---
title: Swift 初探
date: 2020-01-05 15:39:28
tags: [Swift, iOS]
---

# 前言

　　因为公司打算项目全面转 `Swift`，于是我与同事开始了漫长的 `Swift` 探索之路。初用的时候会很不习惯，因为`Swift`和`Objective-C`从语法上来说是两门完全不同的语言，`OC`延续了`C`系语言一贯的啰嗦，需要些很多的代码才能完善这个类或变量，而这些啰嗦的语句其实也深深的嵌入了项目之中，拖慢整个项目的编译进度，增大安装包（虽然不太明显）。

　　这篇文章也不是`Swift`的教程，而是一名`OC`程序猿转`Swift`时所遇到的困难。

<!-- more -->

# Optional

　　`Swift`是一门更加安全的语言，相信很多和我一样的`OC`开发者在新建项目和变量的时候已经开始习惯了这样的方式

```objc
NS_ASSUME_NONNULL_BEGIN
@interface AAPLList : NSObject <NSCoding, NSCopying>
// ...
- (nullable AAPLListItem *)itemWithName:(NSString *)name;
- (NSInteger)indexOfItem:(AAPLListItem *)item;

@property (copy, nullable) NSString *name;
@property (copy, readonly) NSArray *allItems;
// ...
@end
NS_ASSUME_NONNULL_END

// --------------

self.list.name = nil;   // okay

AAPLListItem *matchingItem = [self.list itemWithName:nil];  // warning!
```

　　其中的 `NS_ASSUME_NONNULL_BEGIN`和`NS_ASSUME_NONNULL_END`还有`nullable`与之对应的`nonnull`就是用来增加安全性的设置。我们以前跑项目时经常遇到某个不应该为`nil`的对象突然变成`nil`的情况，其实这样是非常不安全的，但是在多线程的运行中，我们也不一定知道这个对象是在哪里变成`nil`的，所以这样的声明是非常有必要的。这样我们就可以清楚的知道这个对象是否可以为空，这样在传值的时就可以接收到警告，或者运行时就会崩溃方便定位。

　　`Swift`的解决方式就是使用`Optional`，在`Swift`里我们经常会这样声明变量

```swift
var str: String?
```

　　这个问号就很精髓，它就是让这个没有初始化的变量变成一个`Optional`。`Optional`相当于向上封装了一层，代表~~这个变量有值，或者这个变量没有值（废话）~~，但是和不用`nullable`和`nonnull`的`OC`变量不同，你如果直接打印`str`，那你会发现它并不是`String`类型，而是一个`Optional`的类型，里边有`Optional.Some`和`Optional.None`两种类型，如果我们没有给他一个初值，那`Optional`就会返回`None`来告诉我们这个变量为`nil`，而如果我们想访问这个变量的值，那我们需要做的，就是**解包**。

## 解包

　　常见的解包方式一般有两种，一种就是~~比较暴力~~的`!`和相对科学的`??`语法，而另一种则为**可选绑定**



### ! 与 ??

　　`!`通常用在我们知道这个值是必有的情况下，如果我们使用了`!`但这个值为`nil`，那就会引起崩溃，一般来说如果不是100%确定变量有值，是不推荐使用的。

```swift
var str: String?

let temp = str!
// crash

var str: String = "123"

let temp = str!
// temp = "123"
```

　　另一种`??`是一个比较常规的方式，代表如果这个变量为`nil`，则变量为`??`之后的值。我们在`OC`中也会经常用三段表达式进行类似的判断（比如 `NSString *str = tempStr ?: @""`，而`Swift`里的`??`会使这个表达式更加方便简单。

```swift
var str: String?

let temp = str ?? ""
// temp = ""

var str: String = "123"

let temp = str ?? ""
// temp = "123"
```



### 可选绑定

　　可选绑定是一种比较推荐的解包方法，除了麻烦一点，几乎没什么缺点。

　　它是用一个变量去取`Optional`的值，再根据这个变量的值来进行操作。

　　比较常见的是`if let `和 `guard let  else {return}`方法。多说无益，上代码。

```swift
var str1: String?
var str2: String = "123"

if let safeStr = str, safeStr2 = str2 {
    label1.text = safeStr
    label2.text = safeStr2
}
// label1.text = nil
// label2.text = "123"

guard let safeStr1 = str, safeStr2 = str2 else {return}
label1.text = safeStr1
label2.text = safeStr2
// label1.text = nil
// label2.text = nil

guard let safeStr2 = str2 else {return}
label2.text = safeStr2
// label2.text = "123"
```

　　由代码可见，`if let`方法在它的作用域内，我们可以用解包后安全的值进行操作。而`guard let  else {return}`拿到的安全值可以在这个方法的作用域内进行操作。具体选用哪种就需要根据开发者的需求来定了。

　　还有一种可以将两种方式结合的解包方法，不过极不推荐，至于为什么，看代码就知道了！

```swift
var str1: String?
var str2: String = "123"

if let _ = str2 {
    label2.text = str2! 
}
// label2.text = "123"

if let _ = str1 {
    label2.text = str1!
}
// label2.text = nil

guard let _ = str1 else {return}
label1.text = str1!
// label1.text = nil

guard let _ = str2 else {return}
label2.text = str2!
// label2.text = "123"
```

　　因为`guard else`方法将`nil` 的变量`return`了，所以我们强制解包的时候不会有什么问题。但`if let `就不一样了，因为用了强制解包，根据语义应该是必定有值的，但是却并没有，这对于要写优雅代码的我们来说是个不好的写法。

## 总结

`Optional` 让 `Swift`的变量更加安全了，我们在写代码时就能发现很多变量存在的问题，在运行时也不用担心变量突然为 `nil` 的问题了。



# Initialize

搞过 Swift 开发的同学应该知道，Swift 的初始化可以说是相当严苛的，那苹果这么做的目的是什么呢？

其实就是安全，单纯的安全。我们知道，在 OC 中，init 方法是很不安全的，你永远也猜不到一个 init 方法里到底有没有初始化每个变量。

所以 Swift 有了一套很严格的初始化方法，引入了一套名词——或许以前就有？——`designated `、`required`、`convenience`。



## designated

`designated `关键字主要是指明这个方法是子类必须要调用的方法`比如 cell 里需要调用的initWithStyle:reuseIdentifier:`，虽然这个解释看起来更像`required`，但是`required`其实是来指明子类必须进行重写实现的方法。`designated`保证了父类指定的方法肯定会被子类调用，保证该对象可以进行完整的初始化。



## convenience

`convenience`则是一种旨在“补充”的初始化方法，相信大家都在 OC 中的类中声明了许多 `init` 方法，而这些方法都是调用的一个方法`(其实就是 designated)` ，与那个唯一方法的区别就是少了几个变量的初始化（转为了固定值）。`convenience`就是这样的一个方法，以`convenience`声明的`init`方法必须调用**自己类** (self) 的`designated`。值得一提的是`convenience`方法是不允许子类调用的或重写的，这也保证了`所有类都必须调用父类的 designated 方法完成完整的初始化`。

```swift
class ClassA {
    let A: Int
    init(num: Int) {// designated
        A = num
    }

    convenience init(aNum: Bool) {
        // 调用自己类的 designated 方法
        self.init(num: aNum ? 10000 : 1)
    }
}

class ClassB: ClassA {
    let B: Int

    override init(num: Int) {
        numB = num + 1
        // 必须调用父类的 designated，这里也调用不了 ClassA 的 convenience 方法
        super.init(num: num)
    }
}
```

不过如果你重写了父类`convenience`中调用的`init`方法，那你也可以直接在代码中用父类的`convenience`初始化方法初始化子类。

```swift
let A = ClassB(bigNum: true)
// A.A = 10000 A.B = 10001
```



## required

我们可以将一个初始化设置为`required`来让子类强制重写它。比如我们可以让子类重写父类`convenience`中所需要的`designated`方法，来确保子类也可以用父类的`convenience`方法初始化。

```swift
// 还是上面那个方法，不过这次强制子类实现 init(num:)方法了
class ClassA {
    let A: Int
    required init(num: Int) {// designated
        A = num
    }

    convenience init(aNum: Bool) {
        // 调用自己类的 designated 方法
        self.init(num: aNum ? 10000 : 1)
    }
}

class ClassB: ClassA {
    let B: Int
    // 强制重写，不重写的话会报错
    required init(num: Int) {
        numB = num + 1
        // 必须调用父类的 designated，这里也调用不了 ClassA 的 convenience 方法
        super.init(num: num)
    }
}
```

我们甚至组合使用`convenience`和`required`来让子类强制实现（相当于重新声明一个`convenience`）父类的`convenience`方法。这样做可以保证子类不能直接使用到父类的`convenience`方法初始化。



## 总结

根据上面说的，我们可以总结出初始化的时候应该遵循几个原则：

* 本类必须要有个`designated`来达到初始化的目的

* 本类的所有`convenience`初始化方法，都必须调用自己类的`designated`初始化方法来达到完全的初始化
* 子类的`designated`必须调用父类的`designated`方法以保证父类也初始化完成



