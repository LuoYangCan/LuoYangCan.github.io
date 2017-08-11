---
title: 关于iBeacon
date: 2017-08-06 18:54:02
tags: [iBeacon,iOS]

---



因为最近在做的项目是关于蓝牙连接的项目，前辈在做这个项目的时候，除了用ReactiveCocoa以外，还使用了iBeacon技术。

对于从未接触蓝牙这块的我，感觉打开了新世界大门。



## iBeacon

iBeacon是基于地理位置的微定位技术，使用的是Apple提供的CoreLocation（BLE使用的是CoreBluetooth）。根据名字，应该很清楚，使用iBeacon是需要开启定位的，而使用BLE只需要开启蓝牙。  

## 相关术语

* UUID：UUID是Universally UniqueIdentifier（通用唯一标识符）的缩写，实际上是一个随机字符串。在iBeacon中，UUID通常用于表示顶层标识，如果生成一个UUID给iBeacon设备，那么一个设备检测到你的iBeacon时，它就知道它是在和哪个iBeacon通信了。
* major：用于将相关的beacon标识为一组。
* minor：用于标识特定的beacon设备，每个设备都有唯一的minor编号。

下面用一个商场的例子来解释这三个术语

你用有特定UUID的设备与商场里的UUID设备进行通信，一个商店中的所有设备都会被分配到相同的major编号，应用程序根据major编号，就可以知道你大概在哪个商店。而每个商店的每个beacon设备都有唯一的minor编号，那程序通过这个minor编号，就知道你位于商店的某一个位置



## iBeacon属性

iOS中的ibeacon通信数据有

* （NSUUID）ProximityUUID
* （NSNumber）major
* （NSNumber）minor
* （CLProximity）proximity
* （CLLocationAccuracy）accuracy
*   (NSInteger) rssi

分别含义是：

* proximityUUID、major、minor表示ibeacon的uuid、major、minor
* proximity是Apple提供的几个表示距离的属性
  * CLProximityUnknown-没有数据
  * CLProximityImmediate-十厘米以内
  * CLProximityNear-一米以内
  * CLProximityFar-一米以外
* accuracy表示大约距离
* RSSI表示信号强度

根据属性我们可以看到，Apple的判断方式很有趣，它并不去仔细推断距离，而是使用贴近（Immediate）、一米以内（Near）、一米以外（Far）三种状态。距离在1m以内时，RSSI值基本上成比例减少，而在1米以上时，由于各种因素，RSSI是上下波动状态，所以无法推断距离，判定为Far



## iBeacon方法

Apple在iOS4中增加了地理围栏API，可以用来在设备进出某个区域时获得通知，包括了：

* -startMonitoringForRegion:
* -locationManager:didEnterRegion:
* -locationManager:didExitRegion:

这种检测iBeacon的方式叫做**monitoring**。

用这几种方法可以使程序在后台运行时检测iBeacon，但是只能同时检测20个Region，且不能推测设备与Beacon的距离。





除了使用地理围栏API，Apple还在iOS7中新增加了iBeacon的专用检测方式，也就是**ranging**

通过**CLLocationManager**的方法

* `-startRangingBeaconsInRegion:` 检测特定iBeacon。

当检测到beacon的时候，**CLLocationManager**的delegate 

* `-locationManager：didRangeBeacons:inRegion:`会被调用，通知调用者被检测到的beacons。这个方法会返回一个**CLbeacon**数组，根据里面的**proximity**(上文所提到的属性)就可以判断设备与beacon之间的距离。

## iBeacon行为

根据[美团点评技术团队](https://tech.meituan.com/)的文章，暂时有以下结论

* 检测到beacon的时间跟设备进行蓝牙扫描的时间间隔有关，每当设备扫描时，就能发现iBeacon region的变化。
* 在rangging打开的情况下，设备会每秒钟做一次扫描，也就是说状态更新最多延迟一秒。
* 程序在后台运行，并且monitoring打开的时候，设备可能每隔数分钟做一次扫描。iOS7响应较慢，iOS7.1后有较大改善。
* 如果存在设置`notifyEnterStateOnDisplay=yes`的beacon，iOS会在屏幕从黑屏点亮的时候进行一次扫描。
* 设备重启并不影响iBeacon后台检测的执行
* iOS7中，在多任务界面中杀掉程序会终止iBeacon检测的执行，iOS7.1改变了这一行为，被杀掉的app还可以继续进行iBeacon的检测。



在才接触这个项目的初期，好奇于项目与我事先准备的BLE协议实现有些区别，后来了解到用到了iBeacon技术，起初好奇为什么锁屏点亮和锁屏黑屏有什么区别，以为只是因为亮屏可能会激活后台。后来在了解了iBeacon之后，才知道，还有这种操作。所以啊，我们永远都不能放弃学习~。

### 参考

美团点评技术团队：[iBeacon初探](https://tech.meituan.com/iBeacaon-first-glance.html)

