---
title: 项目里的ReactiveCocoa
date: 2017-07-28 18:59:44
tags:RAC

---



​	最近做了一个蓝牙相关的项目，项目的前辈当时是用的ReactiveCocoa（RAC）做的，才开始接手的时候，看的一头雾水。经过一段时间自学，现在对RAC略知一二。



## FunctionalReactiveProgramming

FRP是一种响应变化的编程范式。![FRP](/img/FRP.png)



就像上面的登录界面，在用户输入用户名和密码之前，登陆按钮是处于无法点击状态的，只有当用户名和密码都被填入一定值的时候，才可以点击登陆按钮。这种一个按钮会由于另外几个控件的改变而改变的联动就是FRP。

## ReactiveCocoa

RAC是github上的一个开源项目，可以说是将响应式编程做到了极致。RAC中，通过**RACsignal**来发送信号以执行各种操作

在这里，[limboy(李忠)](limboy.me) 的文章里写的很好

他把信号比作水龙头，但是水龙头里装的是直径与水龙头内径一样的玻璃球(Value)，这样，玻璃球就是依次出来的（没有并发）。水龙头是关着的，需要有接收方（Subscriber）打开，这样只要有玻璃球(Value)出现，就会自动给接收方(subscriber)。还可以在水龙头上加一个滤嘴(Filter)，不符合的东西也不让过。还可以加一个改动装置，把球改成符合自己的需求（map）。也可以合并多个水龙头(combineLastest:reduce:)，这样只有有一个水龙头出玻璃球，这个新水龙头的接收方就会得到这个球。



比如

```javascript
//filter某个属性满足一定条件才执行。  

  [[RACObserve(self, count) filter:^BOOL(id count) {
  if ([count integerValue] == 5) {            
		return YES;        }
	else{           
	 	return NO;        
	}   
 }]subscribeNext:^(id count) {//上面return YES 才执行   

 NSLog(@"数量为===%@",count);    }];

```



RAC还在**UIButton、UITextFiled**等的Category中添加了很多方法，可以直接设置事件。





## 统一了KVO Event Notification等的处理

> KVO

RAC中监听属性改变不再像KVO中用```-observeValueForKeyPath:ofObject:change:context:```

而是使用block

```javascript
// 只有当名字以'j'开头，才会被记录
[[RACAble(self.username) filter:^(NSString *newName) {
       return [newName hasPrefix:@"j"];
   }]
   subscribeNext:^(NSString *newName) {
       NSLog(@"%@", newName);
   }];

```

> Notification

```javascript
[[NSNotificationCenter defaultCenter]rac_addObserverForName:@"ReceiveData" object:nil] subscribeNext:^(NSNotification * _Nullable x) {
                NSlog(@"%@",x);
   } 
```



## 冷热信号

上面提到的只有subscriber订阅时才生效的信号叫做**冷信号**

有冷信号，自然就有**热信号**

* 热信号是主动的，不管你有没有订阅事件，它会时刻推送。
* 热信号可以有多个订阅者，信号和订阅者可以共享信息，多个订阅者可以在订阅开始时同时接收到这个时间及以后的信号（热信号创建时若没有订阅者，它仍然会进行信号发送），而冷信号多个订阅者订阅时，是将信号完整的分别发送给订阅者。

冷热信号的区分，美团点评技术团队的[细说ReactiveCocoa的冷热信号](https://tech.meituan.com/talk-about-reactivecocoas-cold-signal-and-hot-signal-part-1.html)文章写的非常的好。





因为项目的需要，本人还在不断学习，归纳的东西还不够成熟，希望自己能加油吧。