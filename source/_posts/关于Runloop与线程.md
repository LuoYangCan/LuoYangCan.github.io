---
title: Runloop与线程
date: 2017-08-23 16:24:37
tags: [RunLoop,iOS] 
---

## RunLoop

​	RunLoop是每一个iOS程序员应该都听过的一个名字，翻译过来大概是叫运行循环，在iOS攻城狮们的开发初期，几乎见不到RunLoop的身影。

​		<!-- more -->

​	但它其实无处不在，最简单的例子就是Objective-C中的main函数。

```objective-c
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

在UIApplicationMain中，就含有一个Runloop，是系统启动时创建的Runloop。

它有这么几个作用：

* 保证App程序不退出
* 监听用户行为事件
* 监听时钟事件
* 监听网络事件
* 渲染UI

如果没有事件发生，Runloop则会进入休眠状态。

## 监听事件

**监听NSTimer**

```objective-c
    NSTimer * timer = [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(updataTimer) userInfo:nil repeats:YES];

    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];

//相当于上面两句
   [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(updataTimer) userInfo:nil repeats:YES];
```

上面的代码是创建一个Timer，再通知Runloop每隔1s执行一次updataTimer方法

代码看上去虽然没什么问题，但是我们可以发现一个现象：如果我们在当前的Controller中添加了UI控件，当我们做 UI事件（触摸，拖动）时，我们可以发现每隔一秒执行方法的Timer突然停止了，当我们做完这些操作时，Timer又恢复了。

这一现象的出现，就牵扯到Runloop的模式了，也就是`[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];`中的Mode。



RunLoop有五种模式，分别是

* NSDefaultRunLoopMode：默认 Mode，主线程就是在这个 Mode 下运行(默认情况下运行)
* UITrackingRunLoopMode：UI Mode，优先级最高，用于监听UI事件，当发生UI事件时，这个Mode的Runloop优先调用
* NSRunLoopCommonModes：占位 Mode，其实不是一种真正的 Mode ，但在这一模式下，默认Mode和UI Mode都可以被调用（不会因UI操作卡住Timer操作）
* UIInitializationRunLoopMode：在刚启动 App 时进入的第一个 Mode，启动完成后就不再使用。
* GSEventReceiveRunLoopMode：接受系统事件的内部 Mode

在五种模式中，作为开发者，最常用的其实也就前三种模式。

上面的几个方法中，我们的Runloop为`NSDefaultRunLoopMode`，在这种情况（默认模式）下，当发生UI事件时，系统会优先调用`UITrackingRunLoopMode`而不去管默认模式，所以才造成了Timer不执行的情况。

如果我们将`NSDefaultRunLoopMode`改为`NSRunLoopCommonModes`就可以解决问题。

**那么为什么苹果工程师要分UI模式和Default模式呢？**

其实很简单，有耗时操作的存在，当我们在Timer中执行耗时操作时（例如sleep等），如果用占位模式，那么当我们对UI进行操作时，就会回调Timer的方法，因为是耗时操作，就会将界面卡住。

**那么怎么既让我们在进行UI操作的时候执行回调，又不卡住界面呢？**

其实更简单，因为App中的线程不止主线程一个，在苹果漫长的开发中，苹果工程师将UI界面放在了主线程单线程执行，所以，只要我们把耗时操作放到子线程执行，就不会再出现卡住的情况了



## 线程与RunLoop

我们先创建一个自定义线程类，来重写它的-dealloc方法

```objective-c
//LYC_Thread.h
-(void)dealloc{
    NSLog(@"线程结束");
}
```

再进行线程创建

```objective-c
//ViewController.m
#import "LYC_Thread.h"
@interface ViewController ()
@property (nonatomic,strong) LYC_Thread *thread;        /**< 线程  */
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _thread = [[LYC_Thread alloc]initWithBlock:^{
            NSTimer * timer = [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(updataTimer) userInfo:nil repeats:YES];
            [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
        NSLog(@"线程执行");
    }];
    
    [_thread start];
}


- (void)updataTimer {
    NSLog(@"耗时操作");
    [NSThread sleepForTimeInterval:1.0];
    NSLog(@"执行完毕");
}
```

乍一看似乎没有问题，然而当我们运行时发现，操作台的打印效果是这样

![操作台](/img/Thread.png)

通过几个方法，我们可以看到，`NSThread`并没有被释放，但是却并没有执行耗时操作，这是为什么呢？

这是因为这个`NSThread`只是一个对象，而不是线程的本身。

线程是CPU去调用的，CPU负责在线程池里拿出一条线程去执行`NSThread`的任务，一旦结束，线程便没有了。

所以我们要让线程长期存在，并不是去强引用`NSThread`，而是让NSThread有执行不完的任务，这样，线程才会一直存在。

于是我们加入一个死循环在thread中并取消对`LYC_Thread`的强引用

```objective-c
//ViewController.m
- (void)viewDidLoad {
    [super viewDidLoad];
        
    LYC_Thread *thread = [[LYC_Thread alloc]initWithBlock:^{
            NSTimer * timer = [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(updataTimer) userInfo:nil repeats:YES];
        while (true) {
            
        };
            [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
        NSLog(@"线程执行");
    }];
    
    [thread start];
}
```

此时控制台什么也没有打印，也就是说NSThread并没有被释放，也证明了线程并没有被回收



但是，我们加入死循环时，是没有加入事件的。那如果我们在加入死循环时还想加入事件，怎么办呢？

前面我们说到，`RunLoop`的作用相当于一个死循环，而且`RunLoop`还可以监听各种事件。

所以，实现这种需求，`RunLoop`最为合适。



其实每一条**线程**里都默认有一个`RunLoop`，只不过默认不开启。我们可以使用`        [[NSRunLoop currentRunLoop] run];`语句对`RunLoop`进行开启。

开启之后，我们的控制台就会输出这样的信息：

![控制台信息](/img/控制台.png)

我们可以发现，"线程执行"语句没有输出，也证明了`RunLoop`相当于是一个死循环

## 后记

除了开启和运用RunLoop，我们还应该知道如何去关闭RunLoop

* 用` [[NSRunLoop currentRunLoop] runUntilDate:]`方法，可以设定循环的时间
* 用`[NSThread exit]`关闭`NSThread`线程对象





另外，其实主线程和子线程差别也没那么多（本质上应该是相同的）。

我们新建一个子线程后，当我们关闭主线程，子线程仍然能够独立运行，只是主线程的UI不再相应了。





前面提到的``在苹果漫长的开发中，苹果工程师将UI界面放在了主线程单线程执行``就是主线程与子线程的区别。

如果我们多线程操作UI，那么就会发生**资源抢夺**情况，如果要解决这种情况，就需要进行上锁操作。

苹果工程师们在`多线程上锁`和`主线程单线程执行`的选择中，选择了后者。

所以UIKit框架下的控件我们都使用`nonatomic`非原子属性修饰。