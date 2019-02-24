---
title: iOS 触摸事件的传递和响应链
date: 2018-08-15 16:45:17
tags: iOS
---



# 写在前面

　　事件在我们日常的 iOS 开发中可以说是非常的常见了，我们的 App 与用户进行交互就是靠的这一系列事件传递和响应，MVC 中的 View 层也常通过用户的一些事件来进行响应从而通过 Controller 更新 Modal。

<!-- more -->



# 事件

iOS 中的事件可以分为：

* 触摸事件
* 加速计事件
* 远程控制事件



我们从触摸事件入手，对 iOS 的响应链机制进行一定的认识了解



## 响应者对象 UIResponder

　　首先我们要知道，不是每一个对象都可以进行事件的响应和处理的。只有继承了 UIResponder 的对象才能接受并处理事件。

　　我们所熟知的 `UIViewController`和`UIView`都是继承自 UIResponder 的。

　　还有一个我们不那么熟悉的`UIApplication` —— 它是应用程序的象征，是 App 启动后创建的第一个对象 ——也是继承自 UIResponder 的。



UIResponder 里有特殊的方法来处理各个事件：

```objc
//触摸事件
//手指开始触摸 view 时，调用以该方法
- (void)touchesBegan:(NSSet<UITouch *> *)touches 
           withEvent:(UIEvent *)event;
//手指在 view 上移动时，调用该方法
- (void)touchesMoved:(NSSet<UITouch *> *)touches 
           withEvent:(UIEvent *)event;
//手指离开 view 时，调用方法
- (void)touchesEnded:(NSSet<UITouch *> *)touches 
           withEvent:(UIEvent *)event;
//离开前，被某个事件打断(如有电话打入)时，调用该方法
- (void)touchesCancelled:(NSSet<UITouch *> *)touches 
               withEvent:(UIEvent *)event;

//加速计事件
- (void)motionBegan:(UIEventSubtype)motion 
          withEvent:(UIEvent *)event;

- (void)motionEnded:(UIEventSubtype)motion 
          withEvent:(UIEvent *)event;

- (void)motionCancelled:(UIEventSubtype)motion 
              withEvent:(UIEvent *)event;
//远程控制事件
- (void)remoteControlReceivedWithEvent:(UIEvent *)event;
```

从泛型可以看到，`touches` 里面存的是 UITouch 对象，那这个 UITouch 对象又是什么呢？

### UITouch

UITouch 对象保存着跟手指相关的信息（位置、时间等等），一根手指对应一个 UITouch 对象，手指移动的时会更新此对象，手指离开时，会销毁 UITouch 对象。

UITouch 对象有以下属性和方法：

```objc
//有关获取触摸位置
/** 触摸时的 view 或 window */
@property(nonatomic, readonly, strong) UIView *view;
@property(nonatomic, readonly, strong) UIWindow *window;
/** 确定触摸大小的近似值 */
@property(nonatomic, readonly) CGFloat majorRadius;
/** 确定 majorRadius 的准确性           *
**  Radius 加上这个值获得最大的触摸半径    *
**  Radius 减去这个值获得最小的触摸半径    */
@property(nonatomic, readonly) CGFloat majorRadiusTolerance;

//相对 view 的坐标位置
//view 为nil 时，返回相对 UIWindow 的位置
- (CGPoint)locationInView:(UIView *)view;
//前一个 Touch 的位置
- (CGPoint)previousLocationInView:(UIView *)view;

//触摸的一些属性

//点按屏幕的次数，判断是单双击或者更多的点击
@property(nonatomic, readonly) NSUInteger tapCount;
//触摸发生时的时间或最后一次突变的时间
@property(nonatomic, readonly) NSTimeInterval timestamp;
//触摸的状态
@property(nonatomic, readonly) UITouchPhase phase;
//触摸类型 (支持3D Touch)
@property(nonatomic, readonly) UITouchType type;
//点按力量，1.0 表示平均触摸的力量
@property(nonatomic, readonly) CGFloat force;
//触摸时可能的最大力量
@property(nonatomic, readonly) CGFloat maximumPossibleForce;
//用来支持 Apple Pencil 为 0 时表示笔与表面平行，当笔垂直于表面时，值为 Pi/2
@property(nonatomic, readonly) CGFloat altitudeAngle;

//返回方位角
//以笔尖为原点，在屏幕上做 x/y 轴，笔帽(尾部)指向屏幕正 x 轴时，值为 0
//围绕笔尖顺时针摆动时，方位角增大
- (CGFloat)azimuthAngleInView:(UIView *)view;
//返回单位矢量
- (CGVector)azimuthUnitVectorInView:(UIView *)view;
```

### UIEvent

UIEvent 记录着时间产生的时刻和类型，一个事件对应一个 UIEvent

```objc
//事件产生时间
@property(nonatomic, readonly) NSTimeInterval timestamp;
//类型
@property(nonatomic, readonly) UIEventType type;
@property(nonatomic, readonly) UIEventSubtype subtype;
```



　　当用两根手指同时触摸一个 view 的时候，那 view 只会调用一次`touchesBegan:withEvent:`方法，touches 参数中装着两个 UITouch 对象。如果手指一前一后触摸同一个 view，那就会调用两次`touchesBegan:withEvent:`方法。由此可以看出，根据 UITouch 个数就可以判断是单点还是多点触摸。

# 事件产生和传递

## 事件产生

　　发生触摸事件之后，该事件会被加入到一个由 UIApplication 管理的事件队列中。

　　UIApplication 会从队列中取出最前面的时间，并分发下去。一般来说它会把事件给keyWindow 

　　keyWindow 会找一个最合适的 view 来处理这个事件

　　找到这个 view 之后会调用该 view 的 touches 方法做事件处理。

## 事件传递

　　触摸事件的传递是从父控件到子控件的，大概就是 UIApplication -> window -> superview -> view

　　当事件传递找上门的时候，view 会进行如下步骤来配合

1. 判断自己是否能接收触摸事件，能的话就继续，不能的话终止
2. 判断触摸点是不是在自己身上，是的话就继续，不是的话终止
3. 从后往前遍历子控件，再重复前两个步骤
4. 如果没有子控件，那就返回自己，自己就是最佳人选！



UIView 不接受触摸事件的话一般是一下三种情况

* userInteractionEnabled = NO
* hidden = YES
* alpha < 0.01



UIView 提供了两个方法来寻找最合适的 view

```objc
// 用来寻找最合适的View处理事件，只要一个事件传递给一个控件就会调用控件的hitTest方法，参数point 表示方法调用者坐标系上的点
- (UIView *)hitTest:(CGPoint)point 
          withEvent:(UIEvent *)event;

// 用来判断当前这个点在不在方法调用者上，点必须在方法调用者的坐标系中，判断才会准确
- (BOOL)pointInside:(CGPoint)point 
          withEvent:(UIEvent *)event;
```



### hitTest: withEvent

　　只要事件传给一个控件，这个控件就会调用他的`hitTest: withEvent:`方法。这个方法通过调用每个 view 的`pointInside:withEvent:`判断哪个子视图应该接受触摸事件。返回 nil 的时候代表该控件不是最适合的 view，于是系统会去遍历前一个子控件或者直接返回父控件。

　　所以我们可以通过重写`hitTest: withEvent:`返回指定 view，从而达到拦截事件的效果。

Like This

![重写HitTest](/img/hitTest.jpg)

接下来，我们图解一下传递过程

![找到最合适控件](/img/消息传递.png)

当我们点击了蓝色的 view

过程会是：UIApplication -> UIWindow -> 白色 view -> 橙色 view -> 红色 view(返回nil) ->蓝色 view

点击黄色 view 的时候

过程会变成：UIApplication -> UIWindow -> 白色 view -> 橙色 view -> 红色 view (返回 nil) -> 蓝色 view -> 黄色 view



# 事件响应

　　事件的响应是在事件从 UIApplication 传递到 UIView 之后对事件进行处理的响应。默认的做法就是将事件顺着**响应者链条**往上传（甩）递（锅）。

## 响应者链条

　　响应者链条相对于事件传递链的话，可以说是完全相反的，他是由最上面的 View 开始，一直到 UIApplication 的，传递过程中的对象必须是继承自 UIResponder 的响应者对象。

　　响应者链传递过程：

1. 如果当前 view 是 ViewController 的 View，那么控制器就是上一个响应者，事件就传递给 ViewController 。如果不是，那么父视图就是上一个响应者，事件传给父视图
2. 最顶层的视图都不能处理，那就扔给 window
3. 如果 window 也不处理，就丢给 UIApplication
4. UIApplication 也不管的话，就把该事件丢弃。

大致关系可以用下图表示：

![响应者链](/img/响应者链条.png)



# 总结

　　事件处理的流程可以归纳为：

1. 触摸屏幕产生时间后，触摸事件会被添加到 UIApplication 的事件队列中
2. UIApplication 取出最前面的事件，传给 keyWindow
3. keyWindow 找一个最合适的视图来处理事件
4. 最合适的视图调用 touches 方法处理事件
5. touches 默认把事件顺着响应者链往上甩锅
6. 如果 UIApplication 都不能处理这个事件，就丢弃。



所以我们可以通过重写 touches 方法来达到一个事件让多个对象处理的效果。 