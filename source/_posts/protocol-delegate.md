---
title: protocol & delegate
date: 2019-02-24 15:49:44
tags: [iOS]
---

# 前言

　　大学四年好快就过去了，毕业之前去公司实习，发现了自己对代理的一些概念还不理解，遂写了这篇文章。



<!-- more -->



# 关于 protocol & delegate

　　iOS 的 protocol 和 Android 的有一些不一样，因为 iOS 没有多继承，所以很多时候都是用 protocol 来实现。

　　而 delegate 是一种代理的模式，他相当于一个人把一些事情代理给其他人做。比如 A 控制器中有一个值是基于**代理**方法而改变的，当我们从 A 切到 B 控制器时，B 使用了 A 的 `delegate` 将值进行了更改，那当我们回到 A 时，会发现那个值已经改变。



## iOS 中的 protocol



　　一个 protocol 会定义一套接口，接口有两种：

* @required: 必须实现的方法
* @optional: 可选的实现方法



　　我们常常会接触到的 tableViewDataSource 里就有 protocol ，他在里面定义了一些方法让你去实现。

![DataSource](/img/tableViewDataSource.png)

我们可以看到我们用 tableView 的时候必须实现的

```objc
//每个 cell 的创建
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath;

//每个 section 的行数
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section;
```

都是在`@required` 里的，如果我们不实现他，那 tableView 就会空无一值。



而

```objc
// section 个数
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView;   

//header 名字
- (nullable NSString *)tableView:(UITableView *)tableView titleForHeaderInSection:(NSInteger)section;  

//footer 名字
- (nullable NSString *)tableView:(UITableView *)tableView titleForFooterInSection:(NSInteger)section;

……
```

等等是在`@optional`里的，我们是可以选择不去实现他们的。



## delegate

　　我们继续用`tableView`举例，我们使用`tableView`的时候，相当于调用了`UITableView`这个类，让他来帮助我们生成一个 View。

　　但是这个 tableView 并不知道我们需要多高多宽，什么样子的 View，于是他需要我们进行传值。我们用来把值传给它的类就是一个 delegate 。

　　当然，我们经常用 self 来在本控制器下控制 tableView 。



<img src="/img/tableView_nothing.png" width="375" hegiht="650">



# 自定义 delegate

　　看过了系统的一些`delegate`，其实我们也是可以实现代理的，而且使用代理会有许许多多的好处。

　　我们以一个实例来说明，这个例子的逻辑关系基本如下：

![大概思路](/img/delegate_swdt.png)



　　现在让我们来实现他吧

```objc
//DelegateController.h

@class DelegateController;
@protocol TestDelegate <NSObject>

//必须实现
@required

//可选
@optional

-(void)DelegateTest:(DelegateController *)vc;

@end
```

我们先在`.h`文件中创造`delegate`，并声明他的`@required`和`@optional`。

```objc
//DelegateController.h

@interface DelegateController : UIViewController

@property (nonatomic, weak) id<TestDelegate> delegate;        /**< delegate  */

@end
```

然后我们再声明一个`delegate`，之后我们就可以在`DelegateController`里使用`delegate`的方法了

```objc
//DelegateController.m

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    self.view.backgroundColor = [UIColor blueColor];
}

//点击改变背景颜色
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    if ([self.delegate respondsToSelector:@selector(DelegateTest:)]) {
        [self.delegate DelegateTest:self];
    }
}

```

当然，如果我们现在 run 的话，不会有任何的反应，因为`delegate`里的方法还没有进行实现。

我们还得这样

```objc
//mainController.h
- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor whiteColor];
    [self setupDelegateView]; //delegate

}


//delegate实现
-(void)DelegateTest:(DelegatetestVC *)vc{
    vc.view.backgroundColor = [UIColor orangeColor];
    NSLog(@"\n delegate Test Success!");
}

//切到 DelegateController 页面
-(void)setupDelegateView{
    DelegateController *vc = [[DelegatetestVC alloc]init];
    //将 delegate 指定为自己
    vc.delegate = self;
    //延时一秒切换
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        [self presentViewController:vc animated:YES completion:nil];
    });

}

```



 效果大概就是这样啦

<iframe height = 640 width = 360 src = "/img/delegate_demo.gif">



## delegate 的循环引用

　　细心的朋友应该发现了，我们声明 `delegate`的时候用的是`weak`属性

```objc
//weak 属性
@property (nonatomic, weak) id<TestDelegate> delegate;  
```

这是因为我们使用`delegate`的时候常常会这样

```objc
//将 delegate 指定为自己
self.vc.delegate = self;
```

这就让引用变成了`self ->  vc -> self` ，所以如果这时候用`strong`就导致循环引用。

说到循环引用就涉及到 ARC 和内存管理了，可以看我[这篇文章](https://luoyangcan.top/2017/09/13/iOS_ARC/)

当然，我们也可以将代理指定为其他类，那么我们就需要使用`strong`来修饰`delegate`了



# delegate 的一些应用

　　当我们需要实现**工厂模式**的时候，我们可以使用`delegate`，先抽象出一个类声明`delegate`，再在具体的工厂类中实现他，最后我们调用的时候可以将`delegate`设置成我们需要的具体工厂，这样就可以实现工厂模式啦。

# 参考

[Apple](developer.apple.com)