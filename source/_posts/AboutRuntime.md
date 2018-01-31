---
title: Objc 的消息发送机制与 Runtime
date: 2017-09-02 12:19:55
tags: [iOS,runtime]
---



曾觉得 iOS 很好学，也想着学一段时间就可以精通这门语言，但是随着开发的越来越深入，才能意识到，iOS 绝不像外表这样简单，他的内涵真是太深了，感觉自己真是一个什么也不知道的 Objc 小白。

Runtime 和消息发送机制是理解 iOS 运行过程避不开的一道坎，虽然平时很少用，但是却是我们 Objc 程序员需要了解的。



# Runtime

---

因为Objc是一门动态语言，所以它总会在运行时（而不是编译时）进行工作。所以光有一个编译器是不够的，还需要一个运行时系统（runtime system）执行编译后代码。这便是Runtime系统，它是整个Objc运行框架的基石。

## Objc与Runtime的交互

---

**objc从三种不同的层级上与Runtime系统交互，分别是**：



### Objective-C 源代码

部分情况下，runtime都是系统在幕后执行，我们只需要在前台好好写Objc代码就行。

消息执行会使用到一些编译器为实现动态语言特性而创建的数据结构和函数。

**Objc中的类、方法和协议等在runtime中都由一些数据结构定义**





### NSObject的方法

Cocoa中大多数类都继承于`NSObject`类，所以也就继承了它的方法（NSProxy除外）。

NSObject中有许多的方法，自然也有许多作用，比如

* 抽象接口作用，比如`description`方法需要重载它并为你定义的类提供描述内容。
* 在运行时获得类的信息并检查一些特性，比如
  * `class`返回对象的类
  * `isKindOfClass:`和`isMemberofClass:`则检查对象是否在指定的类继承体系中。
  * `respondsToSelector:`检查对象能否响应指定消息（是否有指定方法）。
  * `conformsToProtocol`检查对象是否实现了指定协议方法
  * `methodForSelector:`返回指定方法实现的地址

### Runtime的函数

Runtime系统是一个有一系列函数和数据结构组成，具有公共接口的动态共享库。头文件在`/user/include/objc`中。在[Objective-C Runtime Reference](https://developer.apple.com/documentation/objectivec/objective_c_runtime)中有对Runtime函数的详细文档。



## Runtime基础数据结构

---

在一个类似[a someFuc]的方法调用中，编译阶段编译器并不知道someFuc要实现哪一段代码而只是确定了要向接受者发送someFuc消息，只有到运行的时候，才会发送消息进行方法的确定。这里我们可以看一下objc的底层实现。

```objective-c
//main.m
int main(int argc, const char * argv[]){
    @autoreleasepool{
        Person * p = [[Person alloc] init];
    }
}
```

以上函数在底层其实是这样的

![底层](/img/objc_msgsend.jpg)







如上图所示，其实Objc所有方法在底层都会变成一个函数，那就是`objc_msgSend()`。

```objective-c
id objc_msgSend ( id self, SEL op, ...);
```

这里面有两个参数值得注意，一个是`id`，一个是`SEL`，鉴于id比较复杂，我们先讲讲`SEL`

### SEL

它是`selector`在Objc中的表示类型。`selector`是方法选择器，相当于区分各个方法的一个ID，这个ID的数据结构就是SEL

```objective-c
typedef struct objc_selector *SEL;
```

我们可以用Objc编译器命令`@selector()`或Runtime的`sel_registerName`获得一个`SEL`类型的方法选择器。



### id

作为开发者，大家应该对id都不会陌生，它是一个指向类实例的指针

```objective-c
typedef struct objc_object *id;
```

在这之中，`objc_object`是这样的一个结构体

```objective-c
//objc-private.h
struct objc_object {
private:
    isa_t isa;

public:

    // ISA() assumes this is NOT a tagged pointer object
    Class ISA();

    // getIsa() allows this to be a tagged pointer object
    Class getIsa();
    ... 此处省略其他方法声明
}
```

结构体重包含一个`isa`指针，类型为`isa_t`根据`isa`就可以找到对象所属的类。

`isa`指针又涉及到引用计数原理的知识了，这里就不做详尽描述了。

objc_object中又有属性值得我们注意

#### Class

`Class`其实是一个指向`objc_class`结构体的指针

```objective-c
typedef struct objc_class *Class;
```

这个`objc_class`又包含很多方法了

```objective-c
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
    class_rw_t *data() { 
        return bits.data();
    }
    ... 省略其他方法
}
```

我们可以看到`objc_class`继承于`objc_object`，所以我们可以说一个Objc类本来就是一个对象。

为了处理类和对象的关系，runtime创建了一种叫**元类（Meta Class）**的东西，**类对象所属类型就叫元类**，它用来表述类对象本身所具备的元数据。这就是类方法的定义，每个类仅有一个类对象，每个类也只有一个与之相关的元类。

当我们使用类似`[p alloc]`的类方法时，事实上是把这个消息发送给了一个类对象，这个类对象必须是一个元类的实例，而这个元类也是一个**根元类(root meta class)**的实例。所有元类都指向根元类为其超类。所有元类的方法列表都有能够响应消息的类方法。

所以当`[p alloc]`这条消息发给类对象的时候，`objc_msgSend()`会去它的元类里面去查找能够响应消息的方法，如果找到了，然后就对这个类对象执行方法调用。

![关系图](/img/超类关系图.png)

根据上图，我们可以看到方法，类，元类的关系。有趣的是**根元类**的超类是**根类**（根类在实际运用中就是`NSObject`），`isa`指向了自己。

而`NSObject`的超类为`nil`，也就是说它没有超类。

可以看到运行时一个类还关联了它的超类指针(superclass)，类名，成员变量，方法，缓存，还有附属协议。

##### cache_t

```objective-c
struct cache_t {
    struct bucket_t *_buckets;
    mask_t _mask;
    mask_t _occupied;
    ... 省略其他方法
}
```



`_buckets`存储`IMP`。`_mask`和`_occupied`对应`vtable`。

`cache`是优化的一个机制，如果我们实例对象每收到一个消息都去`isa`指向的类方法列表中遍历，那效率就太低了。

所以系统会把调用的方法存到`cache`中，然后在收到消息后优先在`cache`中查找（理论上讲 如果一个方法被调用一次，那它就很有可能在今后还会被调用）。

`bucket_t`中存储了指针与IMP的键值对：

```objective-c
struct bucket_t {
private:
    cache_key_t _key;
    IMP _imp;

public:
    inline cache_key_t key() const { return _key; }
    inline IMP imp() const { return (IMP)_imp; }
    inline void setKey(cache_key_t newKey) { _key = newKey; }
    inline void setImp(IMP newImp) { _imp = newImp; }

    void set(cache_key_t newKey, IMP newImp);
};
```

详细的细节都在`objc-cache.mm`文件中

##### class_data_bits_t

`class_data_bits_t`包含的信息太多了，主要有`class_rw_t`,`retain/release/autorelease/retaincount`和`alloc`等信息。

```objective-c
//objc-runtime-new.h
struct class_data_bits_t {

	// Values are the FAST_ flags above.
	uintptr_t bits;
	class_rw_t* data() {
	   return (class_rw_t *)(bits & FAST_DATA_MASK);
	}
... 省略其他方法
}
```

联系前面的`Class`，我们可以注意到`objc_class`的`data`方法返回的是`class_data_bits_t`的`data`方法，最终返回的是`class_rw_t`，有好几层。

在`class_data_bits_t`里又包含了一个`bits`，这个指针跟不同的`FAST_`前缀的掩码做按位与操作，可获得不同的数据。`bits`在内存中每个位的含义有三种排列顺序：

![class_data_bits_t](/img/class_data_bits_t.png)

64位不兼容中每个宏对应含义如下:

```objective-c
// class is a Swift class
#define FAST_IS_SWIFT           (1UL<<0)
// class's instances requires raw isa
#define FAST_REQUIRES_RAW_ISA   (1UL<<1)
// class or superclass has .cxx_destruct implementation
//   This bit is aligned with isa_t->hasCxxDtor to save an instruction.
#define FAST_HAS_CXX_DTOR       (1UL<<2)
// data pointer
#define FAST_DATA_MASK          0x00007ffffffffff8UL
// class or superclass has .cxx_construct implementation
#define FAST_HAS_CXX_CTOR       (1UL<<47)
// class or superclass has default alloc/allocWithZone: implementation
// Note this is is stored in the metaclass.
#define FAST_HAS_DEFAULT_AWZ    (1UL<<48)
// class or superclass has default retain/release/autorelease/retainCount/
//   _tryRetain/_isDeallocating/retainWeakReference/allowsWeakReference
#define FAST_HAS_DEFAULT_RR     (1UL<<49)
// summary bit for fast alloc path: !hasCxxCtor and 
//   !instancesRequireRawIsa and instanceSize fits into shiftedSize
#define FAST_ALLOC              (1UL<<50)
// instance size in units of 16 bytes
//   or 0 if the instance size is too big in this field
//   This field must be LAST
#define FAST_SHIFTED_SIZE_SHIFT 51
```

在这里除了`FAST_DATA_MASK`是用一段空间储存数据外，其他宏都是用1bit存bool值。

`class_data_bits_t`提供了三个方法用于位操作：`getBit`,`setBits`和`clearBits`

而`FAST_DATA_MASK`的存储区域里面其实就是存储了指向`class_rw_t`的指针

```objective-c
class_rw_t* data() {
   return (class_rw_t *)(bits & FAST_DATA_MASK);
}
```



#### Category

```objective-c
typedef struct category_t *Category;
```

`Category`为现有的类提供了拓展，存储了类别中可以拓展的实例方法、实例属性和类方法、类属性(objc2016新增特性)。

```objective-c
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```

App 启动加载镜像文件的时候，会简介调用到`attachCategories`函数，完成向类中添加`Category`的工作。

#### Method

```objective-c
typedef struct method_t *Method;
```

它存储了方法名，方法类型和方法实现：

```objective-c
struct method_t {
    SEL name;
    const char *types;
    IMP imp;

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};
```

方法名类型为`SEL`，方法类型`types`是个`char`指针，存储着方法的参数类型和返回值类型。

`imp`指向了方法实现，其实是一个函数指针。

#### Ivar

```objective-c
typedef struct ivar_t *Ivar;


struct ivar_t {
    int32_t *offset;
    const char *name;
    const char *type;
    // alignment is sometimes -1; use alignment() instead
    uint32_t alignment_raw;
    uint32_t size;

    uint32_t alignment() const {
        if (alignment_raw == ~(uint32_t)0) return 1U << WORD_SHIFT;
        return 1 << alignment_raw;
    }
};
```

#### IMP

`IMP`在`objc.h`中为：

```objective-c
typedef void (*IMP)(void /* id, SEL, ... */);
```

它就是一个函数指针，由编译器生成。当我们发起一个Objc消息后，最终会执行什么代码，就由这个指针指定。`IMP`这个函数指针指向了方法的实现。

# 感悟

iOS Runtime真是博大精深，这还没走到最深层，就由一大堆底层概念，所以 学习之路漫漫啊。







# 参考

[杨萧玉的博客](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/)