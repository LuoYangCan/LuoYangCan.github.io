---
title: 关于 iBeacon
date: 2017-08-06 18:54:02
tags: [iBeacon,iOS]

---



因为最近在做的项目是关于蓝牙连接的项目，前辈在做这个项目的时候，除了用 ReactiveCocoa 以外，还使用了 iBeacon 技术。

对于从未接触蓝牙这块的我，感觉打开了新世界大门。



## iBeacon

iBeacon 是基于地理位置的微定位技术，使用的是 Apple 提供的 CoreLocation（BLE 使用的是 CoreBluetooth）。根据名字，应该很清楚，使用 iBeacon 是需要开启定位的，而使用 BLE 只需要开启蓝牙。  

## 相关术语

* UUID：UUID是 Universally UniqueIdentifier（通用唯一标识符）的缩写，实际上是一个随机字符串。在iBeacon 中，UUID 通常用于表示顶层标识，如果生成一个 UUID 给 iBeacon 设备，那么一个设备检测到你的 iBeacon 时，它就知道它是在和哪个 iBeacon 通信了。
* major：用于将相关的 beacon 标识为一组。
* minor：用于标识特定的 beacon 设备，每个设备都有唯一的 minor 编号。

下面用一个商场的例子来解释这三个术语

你用有特定 UUID 的设备与商场里的 UUID 设备进行通信，一个商店中的所有设备都会被分配到相同的 major 编号，应用程序根据 major 编号，就可以知道你大概在哪个商店。而每个商店的每个 beacon 设备都有唯一的 minor 编号，那程序通过这个 minor 编号，就知道你位于商店的某一个位置



## iBeacon属性

iOS中的ibeacon通信数据有

* （NSUUID）ProximityUUID
* （NSNumber）major
* （NSNumber）minor
* （CLProximity）proximity
* （CLLocationAccuracy）accuracy
* (NSInteger) rssi

分别含义是：

* proximityUUID、major、minor 表示 ibeacon 的 uuid、major、minor
* proximity 是 Apple 提供的几个表示距离的属性
  * CLProximityUnknown-没有数据
  * CLProximityImmediate-十厘米以内
  * CLProximityNear-一米以内
  * CLProximityFar-一米以外
* accuracy 表示大约距离
* RSSI 表示信号强度

根据属性我们可以看到，Apple 的判断方式很有趣，它并不去仔细推断距离，而是使用贴近（Immediate）、一米以内（Near）、一米以外（Far）三种状态。距离在 1m 以内时，RSSI 值基本上成比例减少，而在1米以上时，由于各种因素，RSSI 是上下波动状态，所以无法推断距离，判定为 Far



## iBeacon方法

Apple在iOS4中增加了地理围栏API，可以用来在设备进出某个区域时获得通知，包括了：

* -startMonitoringForRegion:
* -locationManager:didEnterRegion:
* -locationManager:didExitRegion:

这种检测 iBeacon 的方式叫做 **monitoring**。

用这几种方法可以使程序在后台运行时检测 iBeacon ，但是只能同时检测 20 个 Region，且不能推测设备与 Beacon 的距离。





除了使用地理围栏 API，Apple 还在 iOS7 中新增加了 iBeacon 的专用检测方式，也就是 **ranging**

通过 **CLLocationManager** 的方法

* `-startRangingBeaconsInRegion:` 检测特定 iBeacon。

当检测到 beacon 的时候，**CLLocationManager** 的 delegate 

* `-locationManager：didRangeBeacons:inRegion:`会被调用，通知调用者被检测到的 beacons。这个方法会返回一个 **CLbeacon** 数组，根据里面的 **proximity** (上文所提到的属性)就可以判断设备与 beacon 之间的距离。

## iBeacon行为

根据[美团点评技术团队](https://tech.meituan.com/)的文章，暂时有以下结论

* 检测到 beacon 的时间跟设备进行蓝牙扫描的时间间隔有关，每当设备扫描时，就能发现 iBeacon region的变化。
* 在 rangging 打开的情况下，设备会每秒钟做一次扫描，也就是说状态更新最多延迟一秒。
* 程序在后台运行，并且 monitoring 打开的时候，设备可能每隔数分钟做一次扫描。iOS7 响应较慢， iOS7.1 后有较大改善。
* 如果存在设置`notifyEnterStateOnDisplay=yes`的 beacon ，iOS 会在屏幕从黑屏点亮的时候进行一次扫描。
* 设备重启并不影响 iBeacon 后台检测的执行
* iOS7 中，在多任务界面中杀掉程序会终止 iBeacon 检测的执行，iOS7.1 改变了这一行为，被杀掉的 app 还可以继续进行 iBeacon 的检测。



在才接触这个项目的初期，好奇于项目与我事先准备的 BLE 协议实现有些区别，后来了解到用到了 iBeacon 技术，起初好奇为什么锁屏点亮和锁屏黑屏有什么区别，以为只是因为亮屏可能会激活后台。后来在了解了iBeacon 之后，才知道，还有这种操作。所以啊，我们永远都不能放弃学习~。

### 参考

美团点评技术团队：[iBeacon初探](https://tech.meituan.com/iBeacaon-first-glance.html)

