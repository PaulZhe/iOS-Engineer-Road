[TOC]

# 引言

> 如果你曾经对 ObjC 底层的实现有一定的了解，你应该会知道 Objective-C 对象都是 C 语言结构体，所有的对象都包含一个类型为 isa 的指针，那么你可能确实对 ObjC 的底层有所知，不过现在的 ObjC 对象的结构已经不是这样了。代替 isa 指针的是结构体 isa_t, 这个结构体中"包含"了当前对象指向的类的信息

- 上面这段文字是引用“[draveness](https://github.com/draveness)”大神的话，因为现代PC机已经全面迈入64位机时代，按以前 isa 的存储方式会造成部分空间的浪费，所以在剩余空间存储了些辅助信息。由下图可知：所有继承自 `NSObject` 的类实例化后的对象都会包含一个类型为 `isa_t` 的结构体。不只是**实例**会包含一个 `isa` 结构体，所有的**类**也有这么一个 `isa`。在 ObjC 中 Class 的定义也是一个名为 `objc_class` 的结构体，如下：

![](http://ww4.sinaimg.cn/large/006tNc79ly1g5rg6dmlw2j30r90by40u.jpg)

# 从 NSObject 的初始化了解 isa

## 对象

**一个 Objective-C 对象的内存结构是怎样的？**

如果把类的实例看成一个C语言的结构体（struct），它首先包含的是一个 isa 指针，而类的其它成员变量依次排列在结构体中。排列顺序如下图所示：

![img](https://images2015.cnblogs.com/blog/515763/201703/515763-20170301231230688-35600101.jpg)

继承于NSObject的类所生成的对象在runtime中的表示是这样的：

```objective-c
struct objc_object {
    isa_t isa;
}
```

很简单，就一个isa_t结构体，从名字也可以看出来这个结构体指明了这个对象是什么，也就是所属的类，isa_t结构体的定义如下：

```objective-c
union isa_t {
    Class cls;
    ...
}
(当然不止这么点内容，后面会详细的分析)
```

可以看到这个结构体中有个类型是Class的属性cls，看起来里面应该存有关于这个对象的类的相关信息，看看Class是如何定义的。

## 类

![img](https://upload-images.jianshu.io/upload_images/1345141-1c9a3f46caca5efc.png?imageMogr2/auto-orient/)

```objective-c
typedef struct objc_class *Class;
struct objc_class : objc_object {
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
}
```

**Class就是结构体objc_class，但是objc_class继承于objc_object，那就是说类其实也是一个对象**，只不过比通常我们理解的对象多了一些属性，比如superclass等。

那类的isa指针指向哪里呢？

- 在class之上，还有叫做元类(meta class)的存在，而class的isa指针就是指向对应的meta class。

- **我们都知道class中存储的是描述对象的相关信息，那么相应的meta class中存放的就是描述class相关信息。说的更直白一点，在我们写代码时，通过对象来调用的方法（实例方法）都是存储在class中的，通过类名来调用的方法（类方法）都是存储在meta class中的。**（如果每一个对象都保存了自己能执行的方法，那么对内存的占用有极大的影响）

下面这张图介绍了对象，类与元类之间的关系：

![img](https://upload-images.jianshu.io/upload_images/4670835-0b5ed7e3cd00094a.jpeg)



这张图解释的非常清楚，meta class的isa指向了root meta class(绝大部分情况下root class就是NSObject的meta class,NSObject的meta class的父类指向自身)，root meta class的isa指向自身，isa的链路就是这样了。

当**实例方法**被调用时，它要通过自己持有的 `isa` 来查找对应的类，然后在这里的 `class_data_bits_t` 结构体中查找对应方法的实现。同时，每一个 `objc_class` 也有一个**指向自己的父类的指针** `super_class` 用来查找继承的方法。

### class 类的打印实验

```objective-c
TestObject *testObj = [TestObject new];
NSLog(@"%d", [testObj class] == [TestObject class]);

这个log会输出1
```

看起来有点奇怪，但是只要看一下源代码实现就能理解了。

```objective-c
+ (Class)class {
    return self;
}

- (Class)class {
    return object_getClass(self);
}
```

object_getClass方法最终返回的是isa。所以TestObject调用class方法，返回的是自身；testObj调用class方法，返回的是isa指向的类，也是TestObject。所以上面结果相同就不奇怪了。

## 结构体 isa_t

**在本篇文章中, 我们会以 __x86_64__ 为例进行分析，而不会对两种架构下由于不同的内存布局方式导致的差异进行分析**。

`isa_t` 是一个 `union` 类型的结构体，其中的 `isa_t`、`cls`、 `bits` 还有结构体共用同一块地址空间。而 `isa` 总共会占据 64 位的内存空间（决定于其中的结构体），在 ObjC 源代码中可以看到这样的定义：

```objective-c
#define ISA_MASK        0x00007ffffffffff8ULL
#define ISA_MAGIC_MASK  0x001f800000000001ULL
#define ISA_MAGIC_VALUE 0x001d800000000001ULL
#define RC_ONE   (1ULL<<56)
#define RC_HALF  (1ULL<<7)

union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;

    struct {
        uintptr_t indexed           : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44;
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;
    };
};
```

> 这是在 `__x86_64__` 上的实现，对于 iPhone5s 等架构为 `__arm64__` 的设备上，具体结构体的实现和位数可能有些差别，不过这些字段都是存在的，可以看这里的 [arm64 上结构体的实现](https://github.com/draveness/analyze/blob/master/contents/objc/从 NSObject 的初始化了解 isa.md#arm64)
>

结构体里的一堆东西其实只有`ISA_BITFIELD`这一个宏，上面只是将其拆解开来，具体内容分析如下：


```objective-c
//        uintptr_t nonpointer        : 1;     
在64位系统中，为了降低内存使用，提升性能，isa中有一部分字段用来存储其他信息，0 表示 raw isa，也就是没有结构体的部分，访问对象的 isa 会直接返回一个指向 cls 的指针，也就是在 iPhone 迁移到 64 位系统之前时 isa 的类型。1 表示当前 isa 不是指针，但是其中也有 cls 的信息，只是其中关于类的指针都是保存在 shiftcls 中。
//        uintptr_t has_assoc         : 1;    
  对象含有或者曾经含有关联引用，没有关联引用的可以更快地释放内存
//        uintptr_t has_cxx_dtor      : 1;   
  表示当前对象有 C++ 或者 ObjC 的析构器(destructor)，如果没有析构器就会快速释放内存
//        uintptr_t shiftcls          : 44;
  /*MACH_VM_MAX_ADDRESS 0x7fffffe00000*/  关于类的指针都是保存在 shiftcls 中。将当前地址右移三位的主要原因是用于将 Class 指针中无用的后三位清除减小内存的消耗，因为类的指针要按照字节（8 bits）对齐内存，其指针后三位都是没有意义的 0。
//        uintptr_t magic             : 6;     
  magic 的值为 0x3b 用于调试器判断当前对象是真的对象还是没有初始化的空间
//        uintptr_t weakly_referenced : 1;     
  对象被指向或者曾经指向一个 ARC 的弱变量，没有弱引用的对象可以更快释放
//        uintptr_t deallocating      : 1;     
  对象正在释放内存
//        uintptr_t has_sidetable_rc  : 1;     
  对象的引用计数太大了，存不下
//        uintptr_t extra_rc          : 8      
  对象的引用计数超过 1，会存在这个这个里面，如果引用计数为 10，extra_rc 的值就为 9
```
## `isa` 的初始化

我们可以通过 `isa` 初始化的方法 `initIsa` 来初步了解这 64 位的 bits 的作用：

```objective-c
inline void 
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
{
    initIsa(cls, true, hasCxxDtor);
}

inline void 
objc_object::initIsa(Class cls, bool indexed, bool hasCxxDtor) 
{ 
    if (!indexed) {
        isa.cls = cls;
    } else {
        isa.bits = ISA_MAGIC_VALUE;
        isa.has_cxx_dtor = hasCxxDtor;
        isa.shiftcls = (uintptr_t)cls >> 3;
    }
}
```

### `indexed` 和 `magic`

当我们对一个 ObjC 对象分配内存时，其方法调用栈中包含了上述的两个方法，这里关注的重点是 `initIsa` 方法，由于在 `initInstanceIsa` 方法中传入了 `indexed = true`，所以，我们简化一下这个方法的实现：

```objective-c
inline void objc_object::initIsa(Class cls, bool indexed, bool hasCxxDtor) 
{ 
    isa.bits = ISA_MAGIC_VALUE;
    isa.has_cxx_dtor = hasCxxDtor;
    isa.shiftcls = (uintptr_t)cls >> 3;
}
```

对整个 `isa` 的值 `bits` 进行设置，传入 `ISA_MAGIC_VALUE`：

```objective-c
#define ISA_MAGIC_VALUE 0x001d800000000001ULL
```

我们可以把它转换成二进制的数据，然后看一下哪些属性对应的位被这行代码初始化了（标记为红色）：

![000](https://imgconvert.csdnimg.cn/aHR0cDovL3d3Mi5zaW5haW1nLmNuL2xhcmdlLzAwNnROYzc5bHkxZzUyMDRtazJ0dGozMWNjMHJrd2dxLmpwZw)

**从图中了解到，在使用 `ISA_MAGIC_VALUE` 设置 `isa_t` 结构体之后，实际上只是设置了 `indexed` 以及 `magic` 这两部分的值。**

### shiftcls

- 在为 `indexed`、 `magic` 和 `has_cxx_dtor` 设置之后，我们就要将当前对象对应的类指针存入 `isa` 结构体中了。

  ```
  isa.shiftcls = (uintptr_t)cls >> 3;
  ```

  > **将当前地址右移三位的主要原因是用于将 Class 指针中无用的后三位清除减小内存的消耗，因为类的指针要按照字节（8 bits）对齐内存，其指针后三位都是没有意义的 0**。

- 使用整个指针大小的内存来存储 `isa` 指针有些浪费，尤其在 64 位的 CPU 上。在 `ARM64` 运行的 iOS 只使用了 33 位作为指针(与结构体中的 33 位无关，Mac OS 上为 47 位)，而剩下的 31 位用于其它目的。类的指针也同样根据字节对齐了，每一个类指针的地址都能够被 8 整除，也就是使最后 3 bits 为 0，为 `isa` 留下 34 位用于性能的优化。

![](http://ww4.sinaimg.cn/large/006tNc79ly1g5rgaov1nuj311j0ldwh7.jpg)

其中红色的为**类指针**，与上面打印出的 `[NSObject class]` 指针右移三位的结果完全相同。这也就验证了我们之前对于初始化 `isa` 时对 `initIsa` 方法的分析是正确的。它设置了 `indexed`、`magic` 以及 `shiftcls`。

### ISA() 方法

- 因为我们使用结构体取代了原有的 isa 指针，所以要提供一个方法 `ISA()` 来返回类指针。

其中 `ISA_MASK` 是宏定义，这里通过掩码的方式获取类指针：

```objective-c
#define ISA_MASK 0x00007ffffffffff8ULL
inline Class 
objc_object::ISA() 
{
    return (Class)(isa.bits & ISA_MASK);
}
```

# 总结

- 最后放一张实验室伙计总结的“神图”来梳理一下这篇博客的知识点

![](http://ww1.sinaimg.cn/large/006tNc79ly1g5rfismazsj31510nmjux.jpg)

### 参考博客：

[analyze](https://github.com/draveness/analyze/blob/master/contents/objc/%E4%BB%8E%20NSObject%20%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96%E4%BA%86%E8%A7%A3%20isa.md)