---
title: 关于@property
date: 2017-10-18 21:59:48
tags: iOS
---

# 前言

不论是初入iOS开发还是已经是老江湖的开发者，想必`@property` 已经成为了我们最熟悉的一个语法。

"属性"(property)作为Objc的一项特性，主要作用就在于封装对象中的数据。Objc对象通常会把所需要的数据保存为各种实例变量。

实例变量一般通过"存取方法"(access method)访问。

获取方法(getter)用于读取变量值

设置方法(setter)用于写入变量值

在正规的Objc编码风格中，存取方法有这严格的命名规范。

正是因为这样的命名规范，所以Objc这门语言才能根据名称自动创建出存取方法。

​	<!-- more -->

# 关于@property

---

想必大家应该都知道或者了解，`@property`语句相当于系统自动为我们生成了`getter`和`setter`方法。

想必大家还应该知道，我们常常在调用实例变量的时候，会出现一个前面带**下划线**的变量，这个变量我们也从未去特意声明过。辣么这个“下划线变量”到底是哪里来的？

没错，就是棒棒的`@property`带来的。

所以我们可以说`@property = ivar(实例变量) + getter + setter`。

比如下面这个经典的例子

```objective-c
@interface Person : NSObject
@property NSString *firstName;
@property NSString *lastName;
@end
```

上面的写法等价于：

```objective-c
@interface Person : NSObject
- (NSString *)firstName;
- (void)setFirstName:(NSString *)firstName;
- (NSString *)lastName;
- (void)setLastName:(NSString *)lastName;
@end
```

## @property属性关键字

属性可拥有的特质分为四类：

* 原子性 —— `nonatomic`

  在默认情况下，由编译器合成的方法会通过锁定机制确保原子性(atomicity)。如果属性具备`nonatomic`特质，则不使用自旋锁。请注意，尽管没有名为"atomic"的特质(如果某属性不是`nonatomic`，那他就是原子的`atomic`)。



* 读写权限 ——`readwrite(读写)`、`readonly(只读)`

* 内存管理 ——`assign`、`strong`、`weak`、`unsafe_unretained`、`copy`

* 方法名 ——`getter = <name>`、`setter = <name>` 

  `getter = <name>`的样式：

  ```objective-c
  @property (nonatomic, getter = isOn) BOOL on;
  ```

  `setter = <name>`一般用在特殊环境下，比如：

  在数据反序列化、转模型的过程中，服务器返回的字段以`init`开头，所以你需要定义一个`init`开头的属性，但默认生成的`getter`和`setter`方法也会以`init`开头。但是编译器会把`init`开头的方法当成初始化方法，而初始化方法只能返回self，所以编译器会报错。

  这时候我们就要用到`setter = <name>`来防止编译器报错

  ```objective-c
  @property(nonatomic, strong, getter=p_initBy, setter=setP_initBy:)NSString *initBy;

  //对关键字特殊说明
  @property(nonatomic, readwrite, copy, null_resettable) NSString *initBy;
  - (NSString *)initBy __attribute__((objc_method_family(none)));
  ```

* 不常用的`nonnull`、`null_resettable`、`nullable`

## ivar

说到ivar，就要涉及到内存管理的知识了

还是那个例子

```objective-c
@interface Person : NSObject
{
    NSString *_firstName;
  	NSInteger _age;
}
@end

  
  
  
@implementation AClass
- (instancetype)init{
    if (self = [super init]){
        _age = 1;
    }
  return self;
}
@end
```

这个Person类被编译之后变成一个描述(arm64)，Person占用24个字节，前8个字节是isa指针，中间八个字节是NSString指针，后八个字节是NSInteger的值

```objective-c
| isa | NSString * _firstName | NSInteger _age |
```

调用`[[Person alloc]init]`的时候，会分配出24个字节的内存出来：

```objective-c
| 0 | 0 | 0 |
```

然后往前8个字节放isa地址

```objective-c
//alloc完成后
| isa | 0 | 0 |
```

然后调用alloc出来的init方法，把\_age的值赋为1，因为\_firstName没有初始化，所以还是0。

```objective-c
//init之后
| isa | 0 | 1 |
```

`isa`指向了这个类的元类，也就是meta，meta里存储了父类/ivar结构/方法等内容，关于这个，在[另一篇文章中](http://luoyangcan.top/2017/09/13/iOS%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86-ARC/)可以看到。

## getter与setter

前言说了

```
Objc对象通常会把所需要的数据保存为各种实例变量。

实例变量一般通过"存取方法"(access method)访问。

获取方法(getter)用于读取变量值

设置方法(setter)用于写入变量值

```

这个观念，几乎都深深的刻在所有程序猿的脑海中。

所以iOS开发者在使用`@property`带来的便利的同时，不能忘记这个重要的想法。

我们所用到的`person.firstName = @"Reus"`其实是一个语法糖，等同于`[person setFirstName:@"Reus"]`。

在过去我们需要声明对应的实例变量`@synthesize person = _person`。

现在，一句`@property`已经可以做到以上所有的步骤了，并且善用`@property`对于内存管理来说，也是一件好事。

# property的那些事

property在runtime中是`objc_property_t`，定义如下：

```objective-c
typedef struct objc_property *objc_property_t;
```

而`objc_property`是一个结构体，包含了`name`和`attributes`。

```objective-c
struct property_t {
    const char *name;
    const char *attributes;
};
```

attributes本质是`objc_property_attribute_t`，定义了properrty的一些属性。

```objective-c
typedef struct{
    const char *name;
 	const char *value;
} objc_property_attribute_t;
```

attributes的具体内容大概包括`类型`，`原子性`，`内存语义`，`对应的实例变量`。

我们定义一个string的property`@property (nonatomic, copy) NSString *string;`，通过`property_getAttributes(property)`获取到attributes并打印，结果为`T@"NSString",C,N,V_string`

T代表类型，C代表Copy，N代表nonatomic，V代表实例变量。

# 自动合成

完成属性定义后，编译器会自动编写访问这些属性所需的方法，此过程叫“自动合成”(autosynthesis)。这个过程是在编译的时候由编译器执行。除了生成getter、setter之外，编译器还要向类中添加适当类型的实例变量，并且在属性名前面加下划线，以此作为实例变量的名字。

可通过`@synthesize`语法来指定实例变量的名字。

```objective-c
@implementation Person
@synthesize firstName = _myFirstName;
@synthesize lastName = _myLastName;
@end
```

[微博@iOS程序犭袁](http://weibo.com/luohanchenyilong/)曾反编译过相关代码，他大致生成了五个东西

* `OBJC_IVAR_$类名$属性名称`：该属性的“偏移量”(offset)，这个偏移量是“硬编码” (hardcode)，表示该变量距离存放对象的内存区域的起始地址有多远。
* setter与getter方法对应的实现函数
* `ivar_list`：成员变量列表
* `method_list`：方法列表
* `prop_list`：属性列表

我们每次增加一个属性，系统都会在`ivar_list`中添加一个成员变量的描述，在`method_list`中增加setter与getter方法的描述，在`prop_list`中增加一个属性的描述，再计算偏移量，给出setter与getter方法对应的实现。在setter方法中从偏移量位置开始赋值，在getter方法中从偏移量开始取值，为了读取正确的字节数，系统偏移量的指针类型进行了强转。

# 写在最后

一般情况下，我们应该多用@property，因为它可以进行某种程度的自动内存管理。但是我们在用@property这样方便的语法时，也千万不能忘记他的本质，这样才更有利于我们对于开发的理解。

## 参考

[segmentfault](https://segmentfault.com/q/1010000000185056)