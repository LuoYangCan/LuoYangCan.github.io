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

## ARC规则

在ARC有效的情况下编译源代码，需要遵守一定的规则：

* 不能使用/retain/release/retainCount/autorelease
* 不能使用NSAllocateObject/NSDeallocateObject
* 须遵守内存管理的方法命名规则
* 不要显式调用dealloc（别用[super dealloc]）
* 使用@autoreleasepool块替代NSAutoreleasePool
* 不能使用NSZone
* 对象型变量不能作为C语言结构体(struct/union)的成员
* 转换id和void *(__bridge)

# 引用计数的储存

---

objc中有些对象如果支持使用TaggedPointer，苹果会直接将其指针值作为引用计数返回。

如果当前设备是64位环境并且使用objective-c 2.0，那么“一些”对象会使用其`isa`指针的一部分控件来存储它的引用计数。

## TaggedPointer

判断当前对象是否在使用TaggedPointer(观察标志位是否为1)

```objective-c
#if SUPPORT_MSB_TAGGED_POINTERS
#   define TAG_MASK (1ULL<<63)
#else
#   define TAG_MASK 1

inline bool 
objc_object::isTaggedPointer() 
{
#if SUPPORT_TAGGED_POINTERS
    return ((uintptr_t)this & TAG_MASK);
#else
    return false;
#endif
}
```

`Objc_object *`看似很陌生，其实它的简写就是我们常看到的`id`(`typedef struct objc_object *id;`)。它的`isTaggedPointer`方法经常会在操作引用计数时用到，因为这决定了存储引用计数的策略。

## isa指针(NONPOINTER_ISA)

用64 bit存储地址太浪费了，于是优化存储方案，用一部分额外空间存储其他内容。

```objective-c
union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;

#if SUPPORT_NONPOINTER_ISA
# if __arm64__
#   define ISA_MASK        0x00000001fffffff8ULL
#   define ISA_MAGIC_MASK  0x000003fe00000001ULL
#   define ISA_MAGIC_VALUE 0x000001a400000001ULL
    struct {
        uintptr_t indexed           : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 30; // MACH_VM_MAX_ADDRESS 0x1a0000000
        uintptr_t magic             : 9;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 19;
#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };

# elif __x86_64__
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x0000000000000001ULL
#   define ISA_MAGIC_VALUE 0x0000000000000001ULL
    struct {
        uintptr_t indexed           : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44; // MACH_VM_MAX_ADDRESS 0x7fffffe00000
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 14;
#       define RC_ONE   (1ULL<<50)
#       define RC_HALF  (1ULL<<13)
    };

# else
    // Available bits in isa field are architecture-specific.
#   error unknown architecture
# endif

// SUPPORT_NONPOINTER_ISA
#endif

};
```

`SUPPORT_NONPOINTER_ISA`用于标记是否支持优化的`isa`指针。

字面意思是`isa`的内容不再是类的指针了，还包含了更多的信息，例如引用计数，析构状态，被其他weak变量引用情况。

```objective-c
// Define SUPPORT_NONPOINTER_ISA=1 to enable extra data in the isa field.
#if !__LP64__  ||  TARGET_OS_WIN32  ||  TARGET_IPHONE_SIMULATOR  ||  __x86_64__
#   define SUPPORT_NONPOINTER_ISA 0
#else
#   define SUPPORT_NONPOINTER_ISA 1
#endif
```

以下是`isa`指针中变量的对应含义

| 变量名               | 含义                                      |
| ----------------- | --------------------------------------- |
| indexed           | 0表示普通的`isa`指针，1表示使用优化，存储引用计数            |
| has_assoc         | 表示该对象是否包含associated object，如果没有，则析构时会更快 |
| has_cxx_dtor      | 表示该对象是否有C++或者ARC的析构函数，如果没有，则析构时更快       |
| shiftcls          | 类的指针                                    |
| magic             | 固定值为0xd2，用于在调试时分辨对象是否未完成初始化。            |
| weakly_referenced | 表示该对象是否有过`weak`对象，如果没有，则析构时更快           |
| deallocating      | 表示该对象是否正在析构                             |
| has_sidetable_rc  | 表示该对象的引用计数值是否过大无法存储在`isa`指针中            |
| extra_rc          | 存储引用计数值减一后的结果                           |

在64位环境下，优化的`isa`指针并不是就一定会存储引用计数，19bit的iOS系统保存引用计数不一定够，这19位保存的是引用计数值减一后的值。

`has_sidetable_rc`的值如果为1，那么引用计数会存储在一个叫`SideTable`的类的属性中。

## 散列表

散列表来存储引用计数具体是用`DenseMap`类来实现，这个类中包含好多映射实例到其引用计数的键值对，并支持用`DenseMapIterator`迭代器快速查找遍历这些键值对。

键值对的格式：

* 键的类型为`DisguisedPtr<objc_object>`，`DisguisedPtr`类是对`objc_object *`指针及其一些操作进行的封装，内容我们可以理解为对象的内存地址
* 值的类型为`__darwin_size_t`。这里保存的值其实就是引用计数减一。

用散列表保存引用计数的优势也很明显，即如果出现故障导致对象的内存块损坏，只要引用计数表没有被破坏，依然可以顺藤摸瓜找到内存块的地址。

## 获取引用计数

在非ARC环境，可以使用`retainCount`方法获取某个对象的引用计数，其会调用`objc_object`的`rootRetainCount()`方法：

```objective-c
- (NSUInteger)retainCount{
    return ((id)self)->rootRetainCount();
}
```

ARC时代除了使用Core Foundation库中的`CFGetRetainCount()`，也可以用Runtime的`_objc_rootRetainCount(id obj)`方法来获取引用计数，此时需要引入`<objc/runtime.h>`头文件。这个函数也是调用`objc_object`的`rootRetainCount()`方法：

```objective-c
inline uintptr_t 
objc_object::rootRetainCount()
{
    assert(!UseGC);
    if (isTaggedPointer()) return (uintptr_t)this;

    sidetable_lock();
    isa_t bits = LoadExclusive(&isa.bits);
    if (bits.indexed) {
        uintptr_t rc = 1 + bits.extra_rc;
        if (bits.has_sidetable_rc) {
            rc += sidetable_getExtraRC_nolock();
        }
        sidetable_unlock();
        return rc;
    }

    sidetable_unlock();
    return sidetable_retainCount();
}
```



`rootRetainCount()`方法对引用计数存储逻辑进行了判断。

除开`TaggedPointer`和`isa`指针的存储方式，会用`sidetable_retainCount()`方法：

```objective-c
uintptr_t
objc_object::sidetable_retainCount()
{
    SideTable *table = SideTable::tableForPointer(this);

    size_t refcnt_result = 1;
    
    spinlock_lock(&table->slock);
    RefcountMap::iterator it = table->refcnts.find(this);
    if (it != table->refcnts.end()) {
        // this is valid for SIDE_TABLE_RC_PINNED too
        refcnt_result += it->second >> SIDE_TABLE_RC_SHIFT;
    }
    spinlock_unlock(&table->slock);
    return refcnt_result;
}
```

`sidetable_retainCount()`方法的逻辑就是先从`SideTable`的静态方法获取当前实例对应的`SideTable`对象，其`refcnts`属性就是之前说的存储引用计数的散列表，这里将其类型简写为`RefcountMap`：

```objective-c
typedef objc::DenseMap<DisguisedPtr<objc_object>,size_t,true> RefcountMap;
```

然后引用计数表中用迭代器查找当前实例对应的键值对，获取引用计数值，并在此基础上加一返回。

我们可以看到有一个`it->second >> SIDE_TABLE_RC_SHIFT`方法将键值对的值做了向右移位的操作

```objective-c
#ifdef __LP64__
#   define WORD_BITS 64
#else
#   define WORD_BITS 32
#endif

// The order of these bits is important.
#define SIDE_TABLE_WEAKLY_REFERENCED (1UL<<0)
#define SIDE_TABLE_DEALLOCATING      (1UL<<1)  // MSB-ward of weak bit
#define SIDE_TABLE_RC_ONE            (1UL<<2)  // MSB-ward of deallocating bit
#define SIDE_TABLE_RC_PINNED         (1UL<<(WORD_BITS-1))

#define SIDE_TABLE_RC_SHIFT 2
#define SIDE_TABLE_FLAG_MASK (SIDE_TABLE_RC_ONE-1)RefcountMap
```

可以看出第一个bit表示该对象是否有过`weak`对象。

第二个bit表示对象是否正在析构。

第三个bit开始才是存储引用计数数值的地方。

所以要向右移两位。

可以用`SIDE_TABLE_RC_ONE`对引用计数+1和-1

用`SIDE_TABLE_RC_PINNED`来判断是否引用计数值有可能溢出。



`_objc_rootRetainCount(id obj)`对于已释放的对象以及不正确的对象地址，有时也返回 “1”。它所返回的引用计数只是某个给定时间点上的值，该方法并未考虑到系统稍后会把自动释放吃池清空，因而不会将后续的释放操作从返回值里减去。clang 会尽可能把 NSString 实现成单例对象，其引用计数会很大。

如果使用了 TaggedPointer，NSNumber 的内容有可能就不再放到堆中，而是直接写在宽敞的64位栈指针值里。其看上去和真正的 NSNumber 对象一样，只是使用 TaggedPointer 优化了下，但其引用计数可能不准确。

**SideTable**

这里提一下`SideTable`，它用于管理引用计数表和`weak`表，并使用`spinlock_lock`自旋锁来防止操作表结构时可能的竞态条件。它用一个64*128大小的`uint8_t`静态数组保存所有`SideTable`实例，提供三个公有属性

```objective-c
spinlock_t slock; //保证原子操作
RefcountMap refcnts; //保存引用计数的散列表
weak_table_t weak_table;//保存weak引用的全局散列表
```

还有一个工厂方法

```objective-c
static SideTable *tableForPointer(const void *p)
/** 根据对象的地址在buffer中寻找对应的SideTable实例 **/
```

`weak`表的作用是在对象执行`dealloc`的时候将所有指向该对象的`weak`指针的值设为`nil`，避免悬空指针。

```objective-c
//weak表的结构
struct weak_table_t{
    weak_entry_t *weak_entries;
  size_t num_entries;
  uintptr_t mask;
  uintptr_t max_hash_displacement;
};
```

苹果用一个全局`weak`表保存所有`weak`引用，将对象作为键，`weak_entry_t`作为值。`weak_entry_t`中保存了所有指向该对象的`weak`指针。

## 修改引用计数

### retain和release

非ARC下，可用`retain`和`release`方法对引用计数进行加一减一操作，它们分别调用了`_objc_rootRetain(id obj)`和`_objc_rootRelease(id obj)`函数。后两者在ARC下也可以使用。

最后这两个函数会调用`objc_object`的两个方法：

```objective-c
inline id 
objc_object::rootRetain()
{
    assert(!UseGC);

    if (isTaggedPointer()) return (id)this;
    return sidetable_retain();
}

inline bool 
objc_object::rootRelease()
{
    assert(!UseGC);

    if (isTaggedPointer()) return false;
    return sidetable_release(true);
}
```

这个实现和获得引用计数类似，先是看是否支持`TaggedPointer`，否则就用`SideTable`里的`refcnts`。

`sidetable_retain()`将引用计数加一后返回对象。

`sidetable_release()`返回是否要执行`dealloc`方法

```objective-c
沙漠中怎么会有泥鳅  11:14:20
bool 
objc_object::sidetable_release(bool performDealloc)
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.indexed);
#endif
    SideTable *table = SideTable::tableForPointer(this);

    bool do_dealloc = false;

    if (spinlock_trylock(&table->slock)) {
        RefcountMap::iterator it = table->refcnts.find(this);
        if (it == table->refcnts.end()) {
            do_dealloc = true;
            table->refcnts[this] = SIDE_TABLE_DEALLOCATING;
        } else if (it->second < SIDE_TABLE_DEALLOCATING) {
            // SIDE_TABLE_WEAKLY_REFERENCED may be set. Don't change it.
            do_dealloc = true;
            it->second |= SIDE_TABLE_DEALLOCATING;
        } else if (! (it->second & SIDE_TABLE_RC_PINNED)) {
            it->second -= SIDE_TABLE_RC_ONE;
        }
        spinlock_unlock(&table->slock);
        if (do_dealloc  &&  performDealloc) {
            ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
        }
        return do_dealloc;
    }

    return sidetable_release_slow(table, performDealloc);
}
```

`it->second < SIDE_TABLE_DEALLOCATING`查看存储的引用计数是否为0，这也是为什么之前存储引用计数时存的是**真正的引用计数值减一后的值**，是为了防止负数的产生。

如果查看存储的引用计数为0，则将对象标记为正在析构(`it->second |= SIDE_TABLE_DEALLOCATING`)，并发送`dealloc`消息，返回`YES`。

否则将引用计数减一（`it->second -+ SIDE_TABLE_RC_ONE`)。

Core Foundation库中也提供了增减引用计数的方法。

```objective-c
//CFBridgingRetain
NS_INLINE CF_RETURNS_RETAINED CFTypeRef __nullable CFBridgingRetain(id __nullable X) {
    return (__bridge_retained CFTypeRef)X;
}
//CFBridgingRelease
NS_INLINE id __nullable CFBridgingRelease(CFTypeRef CF_CONSUMED __nullable X) {
    return (__bridge_transfer id)X;
}
```

`CFBridgingRetain`和`CFBridgingRelease`这两个方法本质是使用`__bridge_retained`和`__bridge_transfer`告诉编译器此处需要如何修改引用计数。

除此之外，objc很多实现还是靠Runtime实现的，Objective-C Runtime源码中有些地方明确注明`//Replaced by CF`，意思就是说，这块任务被`Core Foundation`承包了。

# iOS的内存管理

---

引用计数机制是有些复杂，但是设计到内存管理的话，我们其实不必过多纠结引用计数，直接用以下思路来看待会好一些：

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

```objective-c
id
_objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}
+ (id)alloc {
    return _objc_rootAlloc(self);
}
+ (id)new {
    return [callAlloc(self, false/*checkNil*/) init];
}
```

看得出来`alloc`和`new`最终都会调用`callAlloc`，默认使用Objective-C 2.0且忽视垃圾回收和NSZone。

后续调用顺序为：

```objective-c
class_createInstance()
_class_createInstanceFromeZone()
calloc()
```

`calloc()`函数相比于`malloc()`优点在于它将分配的内存区域初始化为0，相当于`malloc()`后再`memset()`方法再初始化一次。

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
  id obj = [[NSObject alloc] init];
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

# 参考

[玉令天下的博客]( http://yulingtianxia.com/blog/2015/12/06/The-Principle-of-Refenrence-Counting/)

