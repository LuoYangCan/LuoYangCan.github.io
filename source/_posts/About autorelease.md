---
title: 奇特的autorelease
date: 2017-09-17 15:06:48
tags: [iOS]
---

# 引言

一说到 Objective-C 的内存管理，就不得不提到 Autorelease。

顾名思义，autorelease 就是自动释放，这看起来很 ARC，但其实它更像 C语言 中的局部变量特性。

```c
{
    int a;
}
/*
	因超出变量作用域
	局部变量 int a 被废弃。
*/
```

一般来说，autorelease 会像 C语言 的局部变量那样对待对象的实例，一旦超出作用域，对象实例的 release 方法就被调用。还有一点就是编程人员可以设定变量的作用域。

使用方法大致如下：

* 生成并持有 NSAutoreleasePool 对象
* 调用已分配对象的 autorelease 实例方法
* 废弃 NSAutoreleasePool 对象

```objective-c
//表示如下
NSAutoreleasePool *pool = [[[NSAutoreleasePool alloc]init]autorealease];
id obj = [[NSObject alloc] init];
[pool drain];
//相当于[obj release]对象被释放
```

但是，为什么说是一般情况下 autorelease 像 C语言 的局部变量一样呢。

这就涉及到 autorelease 的原理机制了

# Autorelease

其实在没有人工添加 Autoreleasepool 的情况下，Autorealease 对象是在当前的`runloop`结束时释放的，而它能释放的原因也是系统在每个 runloop 中都加入了自动释放池 push 和 pop。

```objective-c
__weak id reference = nil;
-(void)viewdidLoad{
    [super viewDidLoad];
  NSString *str = [NSString string WithFormat:@"Reus"];
  //str是一个autorelease对象，设置一个weak的引用来观察它
  reference = str;
}
- (void) viewWillAppear:(BOOL)animated{
    [super viewWillAppear:animated];
  NSLog(@"%@",reference); //Console: Reus
}
- (void)viewDidAppear:(BOOL)animated{
    [super viewDidAppear:animated];
  NSlog(@"%@",reference);//Console: (null)
}
```

因为这个 vc 在 loadView 之后就 add 到了 window 上，所以`viewDidLoad`和`viewWillAppear`是在同一个 runloop 调用的，因此在`viewWillAppear`中，这个 autorelease 的变量依然有值。

当然我们也可以手动干预

```objective-c
-(void)viewDidLoad{
    [super viewDidLoad];
  @autoreleasepool {
      NSString *str = [NSString stringWithFormat:@"Reus"];
  }
  NSLog(@"%@",str); //Console: (null)
}
```

## AutoreleasePoolPage

ARC 下，我们用`@autoreleasepool{}`来使用一个 AutoreleasePool，随后编译器会转化为这样

```objective-c
void *context = objc_autoreleasePoolPush();
//{}中的代码
objc_autoreleasePoolPop(context);
```

这两个函数都是对`AutoreleasePoolPage`的简单封装，所以自动释放机制核心在于这个类.

`AutoreleasePoolPage`是一个 C++ 实现的类

```c++
//AutoreleasePoolPage
magic_t const magic;
id *next;
pthread_t const thread;
AutoreleasePoolPage * const parent;
AutoreleasePoolPage *child;
uint32_t const depth;
uint32_t hiwat;



class AutoreleasePoolPage{
	static inline void *push() //相当于生成或持有NSAutoreleasePool对象；
	static inline void *pop(void *token) //相当于飞起NSAutoreleasePool类对象
	id *add(id obj) //将对象追加到内部数组中
    void releaseAll()//调用内部数组中对象的release实例方法
}


void *objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}
void objc_autoreleasepoolPop(void *ctxt){
    AutoreleasePoolPage::pop(ctxt);
}
id *objc_autorelease(id obj){
    return AutoreleasePoolPage::autorelease(obj);
}

```

* AutoreleasePool 并没有单独的结构，是由若干个 AutoreleasePoolPage 以`双向链表`的形式组合而成
* AutoreleasePool 是按线程一一对应的(thread指针指向当前线程)
* AutoreleasePoolPage 每个对象会开辟4096字节内存(虚拟内存一页的大小)，除了上面实例变量所占空间。剩下的全是用来存储 autorelease 对象的地址。
* 上面的`id *next`指针作为游标指向栈顶新Add进来的 autorelease 对象的下一个位置
* 一个 AutorelasepoolPage 的空间被占满时，会新建一个 AutoreleasePoolPage 对象，连接链表，后来的 autorelease 对象在新的 page 加入 

向一个对象发送`-autorelease`消息，就是将这个对象加入到当前 AutoreleasePoolPage 的栈顶 next 指针所指向的位置

## 释放

每当进行一次`objc_autoreleasePoolPush`时，runtime向当前的AutoreleasePoolPage中add进一个`哨兵对象`，值为0，`objc_autoreleasePoolPush`的返回值正是这个哨兵对象的地址，被`objc_autoreleasePoolPop(哨兵对象)`作为入参。

所以 Runtime 会

* 根据传入的哨兵对象地址找到哨兵对象所处的 page
* 在当前 page 中将晚于哨兵对象插入的所有 autorelease 发送一次`-release`消息，并向回移动`next`到正确位置

### 嵌套AutoreleasePool

因为上面也讲到了，pop 的时候总是会释放到上次的 push 位置为止，多层嵌套也就是多个哨兵对象，一层一层的释放，互不影响。



# 参考

---

[sunnyxx的博客](http://blog.sunnyxx.com)

