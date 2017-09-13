---
title: iOS内存管理 - ARC
date: 2017-09-13 16:31:46
tags: [iOS,ARC]
---

众所周知，Apple从OS X Lion和iOS 5引入了新的内存管理功能——自动引用计数(ARC)功能。这些功能对于我们开发者说也是需要去了解的一个重要知识点。

# 自动引用计数

------

```官方文档
在Objective-C中采用Automatic Reference Counting机制，让编译器来进行内存管理。
在新一代Apple LLVM编译器中设置ARC为有效状态，就无需在此键入retain和release代码，
这在降低程序崩溃、内存泄漏等风险的同时，很大程度上减少了开发程序的工作量。
编译器完全清楚目标对象，并能立刻释放那些不再被使用的对象。
如此一来，应用程序将具有可预测性，且能流畅运行，速度也将大幅提升。
								--------- Apple
```

ARC的机制可以用开关房间里的灯的事例来说明：

- 进入房间的人需要灯光照明。
- 离开房间的人不需要灯光照明。
- 如果离开房间的人因不需要照明而把灯关掉，那房间里剩下的人则不能得到照明。



解决办法就是使房间还有至少1人的情况下保持开灯，无人时关灯。

为了判断是否还有人在房间里，我们导入计数功能来计算“需要照明的人数”：

- 第一个人进入房间，“需要照明人数”+1，计数值由0变为1
- 之后每有一个人进入房间，“需要照明人数”就+1。
- 每当有人离开房间，“需要照明人数”就-1 。
- 最后一个人下班离开房间时，“需要照明人数”就从1减到了0，所以要关灯。

我们将这个事例套入到objc的中，对应的关系如下：

| 对照明设备所做操作 | 对objc对象所做动作 |
| :-------- | :---------- |
| 开灯        | 生成对象        |
| 需要照明      | 持有对象        |
| 不需要照明     | 释放对象        |
| 关灯        | 废弃对象        |

上述中的`生成`、`持有`、`释放`、`废弃`概念可以对应objc的这些方法：

| 对象操作    | Objective-C方法                 |
| ------- | ----------------------------- |
| 生成并持有对象 | alloc/new/copy/mutableCopy等方法 |
| 持有对象    | retain方法                      |
| 释放对象    | release方法                     |
| 废弃对象    | dealloc方法                     |







---



我们可以用一下思路来看内存管理，不必去纠结引用计数

- 自己生成的对象，自己持有。
- 非自己生成的对象，自己也可以持有。
- 不再需要自己持有的对象时释放。
- 非自己持有的对象无法释放。

## 自己生成的对象，自己持有

使用以下名称开头的方法名意味着自己生成的对象只有自己持有：

- alloc
- new
- copy
- mutableCopy

### alloc/new

```objective-c
//自己生成并持有对象
id obj1 = [[NSObject alloc] init];
//自己持有对象


//自己生成并持有对象
id obj2 = [NSObject new];
//自己持有对象
```

一般来说，`[NSObject new]`与`[[NSObject alloc] init]`是完全一样的。

区别在于`alloc`因将关联对象内存分配到相邻区域从而更加省时省力。

`alloc`也可以用自定义的`init`方法(例如`initWithFrame`)而new只能用默认的init。



### copy

`copy`用基于NSCopying方法约定，由各类实现的`copyWithZone：`方法生成并持有对象的副本。与`copy`方法类似。

`mutableCopy`利用基于`NSMutableCopying`方法约定，由各类实现的`mutableCopyWithZone：`方法生成并持有对象的副本。

区别在于，`copy`方法生成不可变更的对象，而`mutableCopy`生成可变更的对象。类似`NSArray`与`NSMutableArray`类对象的差异。这里还涉及到一个深浅拷贝的知识点。

两个方法虽然生成的是对象的副本，但是同`alloc`、`new`一样在**自己生成并持有对象**这点上没有改变。

## 非自己生成的对象，自己也能持有

用上述方法以外的方法（即用`alloc`、`new`、`copy`和`mutableCopy`以外的方法）取得的对象，因为非自己生成持有，所以自己不是该对象的持有者。

```objective-c
//取得非自己生成并持有的对象
id obj = [NSMutableArray array];
//取得的对象存在，但自己不持有对象
[obj retain];
//自己持有该对象
```

通过`retain`方法，非自己生成的对象跟用`alloc/new/copy/mutableCopy`生成并持有的对象一样名称为了自己所持有的。

## 不再需要自己持有的对象时释放

自己持有的对象，一旦不需要，持有者有义务释放该对象。释放使用`release`方法。

```objective-c
//自己生成并持有对象
id obj = [[NSObject alloc] init];
//自己持有对象
[obj release];
/* 
	释放对象
	
	指向对象的指针仍然被保留在变量obj中
	
	对象一经释放就不可被访问
*/


//取得非自己生成并持有的对象
id obj = [NSMutableArray array];

//取得的对象存在 但自己不持有

[obj retain];

//持有对象

[obj release];

/*
	释放对象
	对象不可再被访问
*/
```

如果要用某个方法生成对象，并将其返还给该方法的调用方，则需要以下方法

```objective-c
- (id) allocObject {
    //自己生成并持有
  id obj = [[NSOBject alloc] init];
  //自己持有对象
  return obj;
}

/*
	原封不动的返回alloc方法生成并持有的对象，就能让调用方也持有该对象。
*/


//取得非自己生成并持有的对象

id obj1 =[obj0 allocObject];

//自己持有对象
```

注意，`allocObject`方法符合`以alloc/new/copy/mutableCopy方法开头`并用驼峰拼写法命名的命名规则。因此它与`alloc`方法生成并持有对象的情况完全相同。

若使取得的对象存在，但自己不持有对象，就需要这样

```objective-c
-(id)Object{
    id obj = [[NSObject alloc] init];
  //自己持有对象
  [obj autorelease];
  //取得对象存在，但不持有
  return obj;
}
```

上例中，我们使用了`autorelease`方法。用该方法，可以使取得的对象存在，但自己不持有对象。

`autorelease`提供这样的功能，使对象在超出指定的生存范围时能够自动并正确地释放(调用`release`)。

使用`[NSmutableArray array]`方法取得谁都不持有的对象，就是通过`autorelease`实现的。

## 无法释放非自己持有的对象

对于用`alloc/new/copy/mutableCopy`方法生成并持有的对象，或是用`retain`方法持有的对象，由于持有者是自己，所以在不需要该对象时需要将其释放。

而由此以外所得到的对象绝对不能释放。倘若在应用程序中释放了非自己持有的对象就会造成崩溃。

```objective-c
/**释放完不再需要的对象后再次释放**/

//自己生成并持有对象
id obj =[[NSObject alloc] init];
//自己持有对象
[obj release];
//对象已释放
[obj release];
//程序崩溃


/**取得对象存在，但自己不持有时释放**/
id obj1 = [obj0 object];
//取得的对象存在，但自己不持有对象
[obj1 release];
//程序崩溃
```

如以上例子，释放非自己持有的对象会造成程序崩溃，因此绝对不要去释放非自己持有的对象。

