[TOC]

# Blocks的实质

在[iOS开发：Blocks探究——基本用法](https://blog.csdn.net/qq_42347755/article/details/100830133)中我们知道了 Blocks 是 **带有局部变量的匿名函数**。但是 Block 的实质究竟是什么呢？

## 准备工作：OC 转 C++ 源码方法

1. 在项目中添加 blocks.m 文件，并写好 block 的相关代码。
2. 打开『终端』，执行 `cd XXX/XXX` 命令，其中 `XXX/XXX` 为 block.m 所在的目录。
3. 继续执行`clang -rewrite-objc block.m`
4. 执行完命令之后，block.m 所在目录下就会生成一个 block.cpp 文件，这就是我们需要的 block 相关的 C++ 源码。

## Blocks 源码概览

下面我们删除掉 block.m 其他无关的代码，只保留 blocks 相关的代码，可以得到如下结果。

- 转换前 OC 代码：

```objectivec
int main () {
    void (^myBlock)(void) = ^{
        printf("myBlock\n");
    };

    myBlock();

    return 0;
}
```

- 转换后 C++ 源码：

```cpp
/* 包含 Block 实际isa指针的结构体 */
struct __block_impl {
    void *isa;
    int Flags;               
    int Reserved;        // 今后版本升级所需的预留区域大小
    void *FuncPtr;      // 函数指针
};

/* Block 结构体 */
struct __main_block_impl_0 {
    // impl：应该是implementation的缩写，指的是block实现部分的结构体
    struct __block_impl impl;
    // Desc：应该是description的缩写，包含 Block 附加信息
    struct __main_block_desc_0* Desc;
    // __main_block_impl_0：Block 构造函数
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

/* Block 函数 */
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    printf("myBlock\n");
}

/* Block 附加信息结构体：包含今后版本升级所需区域大小，Block 的大小*/
static struct __main_block_desc_0 {
    size_t reserved;        // 今后版本升级所需区域大小
    size_t Block_size;    // Block 大小
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)}; //初始化的一个 Block 附加信息结构体的实例

/* main 函数 */
int main () {
    void (*myBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));//返回block构造函数构造出的结构体的指针
    ((void (*)(__block_impl *))((__block_impl *)myBlock)->FuncPtr)((__block_impl *)myBlock);//调用刚赋值给blk变量中的成员函数

    return 0;
}
```

这么看可能不够清楚，下面我们来把这一大块源码进行拆解。

## Block 结构体

我们先来看看 `__main_block_impl_0` 结构体（ Block 结构体）

```cpp
/* Block 结构体 */
struct __main_block_impl_0 {
    // impl：应该是implementation的缩写，指的是block实现部分的结构体
    struct __block_impl impl;
    // Desc：应该是descriptor的缩写(描述符号)，包含 Block 附加信息
    struct __main_block_desc_0* Desc;
    // __main_block_impl_0：Block 构造函数
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
```

从上边我们可以看出，`__main_block_impl_0` 结构体（Block 结构体）包含了三个部分：

1. 成员变量 `impl`;
2. 成员变量 `Desc` 指针;
3. `__main_block_impl_0` 构造函数。

我们先来把这几个部分剖析一下。

###  1. `struct __block_impl impl` 说明

第一部分 `impl` 是 `__block_impl` 结构体类型的成员变量。`__block_impl` 包含了 :

- Block 实际函数指针 `FuncPtr`，`FuncPtr` 指针指向 Block 的主体部分，也就是 Block 对应 OC 代码中的 `^{ printf("myBlock\n"); };` 部分。
- 标志位 `Flags`，在实现block的内部操作时会用到
- 今后版本升级所需的区域大小 `Reserved`
- `__block_impl` 结构体的实例指针 `isa`。在这里我们又见到了熟悉的`isa `指针，由此我们可以看出：**所谓 Block 就是 Objective-C 对象。**

```cpp
/* 包含 Block 实际函数指针的结构体 */
struct __block_impl {
    void *isa;               // 用于保存 Block 结构体的实例指针
    int Flags;               // 标志位，在实现block的内部操作时会用到
    int Reserved;        // 今后版本升级所需的区域大小
    void *FuncPtr;      // 函数指针
};
```

### 2. `struct __main_block_desc_0* Desc` 说明

第二部分 Desc 是指向的是 `__main_block_desc_0` 类型的结构体的指针型成员变量，`__main_block_desc_0` 结构体用来描述该 Block 的相关附加信息：

1. 今后版本升级所需区域大小： `reserved` 变量。
2. Block 大小：`Block_size` 变量。

```cpp
/* Block 附加信息结构体：包含今后版本升级所需区域大小，Block 的大小*/
static struct __main_block_desc_0 {
    size_t reserved;        // 今后版本升级所需区域大小
    size_t Block_size;    // Block 大小
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)}; //初始化的一个 Block 附加信息结构体的实例
```

### 3. `__main_block_impl_0` 构造函数说明

第三部分是 `__main_block_impl_0` 结构体（Block 结构体） 的构造函数，负责初始化 `__main_block_impl_0` 结构体（Block 结构体） 的成员变量。

```cpp
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
}
```

关于结构体构造函数中对各个成员变量的赋值，我们需要先来看看 `main()` 函数中，对该构造函数的调用。

```cpp
  void (*myBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
```

我们可以把上面的代码稍微转换一下，去掉不同类型之间的转换，使之简洁一点：

```cpp
struct __main_block_impl_0 temp = __main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA);
struct __main_block_impl_0 myBlock = &temp;
```

这样，就容易看懂了。该代码将通过 `__main_block_impl_0` 构造函数，生成的 `__main_block_impl_0` 结构体（B lock 结构体）类型实例的指针，赋值给 `__main_block_impl_0` 结构体（Block 结构体）类型的指针变量 `myBlock`。

可以看到， 调用 `__main_block_impl_0` 构造函数的时候，传入了两个参数。

1. 第一个参数：`__main_block_func_0`。

   - 其实就是 Block 对应的主体部分(函数指针)，传入了一个函数指针，利用 （void *）进行了一次强制类型转换。观察这个传入的函数可知，**通过 Blocks 使用的匿名函数实际上被作为简单的C语言函数来处理。**

   - 这个 Block 的主体函数是根据 Block 语法所属的函数名（此处为main）和该 Block 语法在该函数出现的顺序值（此处为0）来给经 clang 变换的函数命名。

   - 这里参数中的 `__cself` 是指向 Block 的值的指针变量，相当于 OC 中的 `self`。

     ```cpp
     /* Block 主体部分函数 */
     static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
             printf("myBlock\n");
     }
     ```

2. 第二个参数：`__main_block_desc_0_DATA`：`__main_block_desc_0_DATA` 包含该 Block 的相关信息。
   我们再来结合之前的 `__main_block_impl_0` 结构体定义。

   - `__main_block_impl_0` 结构体（Block 结构体）可以表述为：

     ```cpp
     struct __main_block_impl_0 {
         void *isa;               // 用于保存 Block 结构体的实例指针
         int Flags;               // 标志位
         int Reserved;        // 今后版本升级所需的区域大小
         void *FuncPtr;      // 函数指针
         struct __main_block_desc_0* Desc;      // Desc：Desc 指针
     };
     ```

   - `__main_block_impl_0` 构造函数可以表述为：

     ```cpp
     impl.isa = &_NSConcreteStackBlock;    // _NSConcreteStackBlock 相当于 class_t 结构体实例。在将 Block 作为OC 对象处理时，关于该类的信息放置于 _NSConcreteStackBlock 中。
     impl.Flags = 0;        // 标志位赋值
     impl.FuncPtr = __main_block_func_0;    // FuncPtr 保存 Block 结构体的主体部分
     Desc = &__main_block_desc_0_DATA;    // Desc 保存 Block 结构体的附加信息
     ```

## Block 实质总结

千言万语汇成一句话：Block 的实质就是对象。在C语言的底层实现里，它是一个结构体。这个结构体相当于`objc_class`的类对象结构体，用`_NSConcreteStackBlock`对其中成员变量`isa`初始化，`_NSConcreteStackBlock`相当于`class_t`结构体实例(在我的理解中就是该 block 实例的元类)。在将 Block 作为OC对象处理时，关于该类的信息放置于`_NSConcreteStackBlock`中。

***

# 截获自动变量值

```objectivec
// 使用 Blocks 截获局部变量值
- (void)useBlockInterceptLocalVariables {
    int a = 10, b = 20;

    void (^myLocalBlock)(void) = ^{
        printf("a = %d, b = %d\n",a, b);
    };

    myLocalBlock();    // 输出结果：a = 10, b = 20

    a = 20;
    b = 30;

    myLocalBlock();    // 输出结果：a = 10, b = 20
}
```

从中可以看到，我们在第一次调用 `myLocalBlock();` 之后已经重新给变量 `a`、变量 `b` 赋值了，但是第二次调用 `myLocalBlock();` 的时候，使用的还是之前对应变量的值，这点在上篇基本用法的博客中已经解释过了，这是由于： **Blocks 变量截获局部变量值的特性。**

- 总的来说，所谓“截获自动变量值”意味着在执行 Block 语法时，Block 语法表达式所使用的自动变量值被保存到 Block 的结构体实例（即 Block 自身）中。
- 不过请注意，Block 语法表达式中没有使用的自动变量不会被追加到结构体中，Blocks 的自动变量截获只针对 Block 中**使用的**自动变量。

我们来看一下对应的 C++ 代码：

```cpp
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    int a;
    int b;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _a, int _b, int flags=0) : a(_a), b(_b) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int a = __cself->a; // bound by copy
    int b = __cself->b; // bound by copy

    printf("a = %d, b = %d\n",a, b);
}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main () {
    int a = 10, b = 20;

    void (*myLocalBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, a, b));
    ((void (*)(__block_impl *))((__block_impl *)myLocalBlock)->FuncPtr)((__block_impl *)myLocalBlock);

    a = 20;
    b = 30;

    ((void (*)(__block_impl *))((__block_impl *)myLocalBlock)->FuncPtr)((__block_impl *)myLocalBlock);
}
```

1. 可以看到 `__main_block_impl_0` 结构体（Block 结构体）中多了两个成员变量 `a` 和 `b`，这两个变量就是 Block 截获的局部变量。 `a` 和 `b` 的值来自与 `__main_block_impl_0` 构造函数中传入的值。

   ```cpp
       struct __main_block_impl_0 {
           struct __block_impl impl;
           struct __main_block_desc_0* Desc;
           int a;    // 增加的成员变量 a
           int b;    // 增加的成员变量 b
           __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _a, int _b, int flags=0) : a(_a), b(_b) {    
               impl.isa = &_NSConcreteStackBlock;
               impl.Flags = flags;
               impl.FuncPtr = fp;
               Desc = desc;
           }
       };
   ```

2. `还可以看出 __main_block_func_0`（保存 Block 主体部分的结构体）中，变量 `a`、`b` 的值使用的 `__cself` 获取的值。
   而 `__cself->a`、`__cself->b` 是通过值传递的方式传入进来的，而不是通过指针传递。这也就说明了 `a`、`b` 只是 Block 内部的变量，改变 Block 外部的局部变量值，并不能改变 Block 内部的变量值。

   ```cpp
       static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
           int a = __cself->a; // bound by copy
           int b = __cself->b; // bound by copy
           printf("a = %d, b = %d\n",a, b);
       }
   ```

> 那么来总结一下：
>
> 在定义 Block 表达式的时候，局部变量使用**『值传递』**的方式传入 Block 结构体中，并保存为 Block 的成员变量。
>
> 而当外部局部变量发生变化的时候，Block 结构体内部对应的的成员变量的值并没有发生改变，所以无论调用几次，Block 表达式结果都没有发生改变。

# Blocks 内改写被截获变量的值的办法

## 更改特殊区域变量值

- 第一种：C语言中有几种特殊区域的变量，允许 Block 改写值，具体如下：
  - 静态局部变量
  - 静态全局变量
  - 全局变量

下面我们通过代码来看看这种情况

- OC 代码：

```cpp
int global_val = 10; // 全局变量
static int static_global_val = 20; // 静态全局变量

int main() {
    static int static_val = 30; // 静态局部变量

    void (^myLocalBlock)(void) = ^{
        global_val *= 1;
        static_global_val *= 2;
        static_val *= 3;

        printf("static_val = %d, static_global_val = %d, global_val = %d\n",static_val, static_global_val, static_val);
    };

    myLocalBlock();

    return 0;
}
```

- C++ 代码：

```cpp
int global_val = 10;
static int static_global_val = 20;

struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    int *static_val;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_static_val, int flags=0) : static_val(_static_val) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int *static_val = __cself->static_val; // bound by copy
    global_val *= 1;
    static_global_val *= 2;
    (*static_val) *= 3;

    printf("static_val = %d, static_global_val = %d, global_val = %d\n",(*static_val), static_global_val, (*static_val));
}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

int main() {
    static int static_val = 30;

    void (*myLocalBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, &static_val));
    ((void (*)(__block_impl *))((__block_impl *)myLocalBlock)->FuncPtr)((__block_impl *)myLocalBlock);

    return 0;

}
```

从中可以看到：

- 在 `__main_block_impl_0` 结构体中，将静态局部变量 `static_val` 以指针的形式添加为成员变量，而静态全局变量 `static_global_val`、全局变量 `global_val` 并没有添加为成员变量。

```cpp
int global_val = 10;
static int static_global_val = 20;

struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    int *static_val;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_static_val, int flags=0) : static_val(_static_val) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
```

再来看一下 Block 主体部分对应的 `__main_block_func_0` 结构体部分。静态全局变量 `static_global_val`、全局变量 `global_val` 是直接访问的，而静态局部变量 `static_val` 则是通过『指针传递』的方式进行访问和赋值。

```cpp
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int *static_val = __cself->static_val; // bound by copy
    global_val *= 1;
    static_global_val *= 2;
    (*static_val) *= 3;

    printf("static_val = %d, static_global_val = %d, global_val = %d\n",(*static_val), static_global_val, (*static_val));
}
```

- 静态变量的这种方法似乎也适用于自动变量的访问，但我们为什么没有这么做呢？

  实际上，在由 Block 语法生成的值 Block 上，可以存有超过其变量域的被截获对象的自动变量。变量作用域结束的同时，原来的自动变量被废弃，因此 Block 中超过变量作用域而存在的变量同静态变量一样，将不能通过指针访问原来的自动变量。

## __block 说明符

### __block修饰局部变量

- 第二种方法是使用`__block 说明符`，更准确的表达方式为`__block 存储域类说明符`。
- `__block 说明符`类似于 `static`、`auto`、`register` 说明符，它们用于指定将变量值设置到哪个存储域中。例如`auto` 表示作为自动变量存储在**栈**中， `static`表示作为静态变量存储在**数据区**中。

```cpp
// 使用 __block 说明符修饰，更改局部变量值
- (void)useBlockQualifierChangeLocalVariables {
    __block int a = 10, b = 20;

    void (^myLocalBlock)(void) = ^{
        a = 20;
        b = 30;

        printf("a = %d, b = %d\n",a, b);    // 输出结果：a = 20, b = 30
    };

    myLocalBlock();
}
```

从中我们可以发现：通过 `__block` 修饰的局部变量，可以在 Block 的主体部分中改变值。

我们来转换下源码，分析一下：

```cpp
struct __Block_byref_a_0 {
    void *__isa;
    __Block_byref_a_0 *__forwarding;
    int __flags;
    int __size;
    int a;
};

struct __Block_byref_b_1 {
    void *__isa;
    __Block_byref_b_1 *__forwarding;
    int __flags;
    int __size;
    int b;
};

struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __Block_byref_a_0 *a; // by ref
    __Block_byref_b_1 *b; // by ref
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_a_0 *_a, __Block_byref_b_1 *_b, int flags=0) : a(_a->__forwarding), b(_b->__forwarding) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    __Block_byref_a_0 *a = __cself->a; // bound by ref
    __Block_byref_b_1 *b = __cself->b; // bound by ref

    (a->__forwarding->a) = 20;
    (b->__forwarding->b) = 30;

    printf("a = %d, b = %d\n",(a->__forwarding->a), (b->__forwarding->b));
}

static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->a, (void*)src->a, 8/*BLOCK_FIELD_IS_BYREF*/);_Block_object_assign((void*)&dst->b, (void*)src->b, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->a, 8/*BLOCK_FIELD_IS_BYREF*/);_Block_object_dispose((void*)src->b, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
    void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
    void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

int main() {
    __attribute__((__blocks__(byref))) __Block_byref_a_0 a = {(void*)0,(__Block_byref_a_0 *)&a, 0, sizeof(__Block_byref_a_0), 10};
    __Block_byref_b_1 b = {(void*)0,(__Block_byref_b_1 *)&b, 0, sizeof(__Block_byref_b_1), 20};

    void (*myLocalBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_a_0 *)&a, (__Block_byref_b_1 *)&b, 570425344));
    ((void (*)(__block_impl *))((__block_impl *)myLocalBlock)->FuncPtr)((__block_impl *)myLocalBlock);

    return 0;
}
```

可以看到，只是加上了一个 `__block`，代码量就增加了很多。

我们从 `__main_block_impl_0` 开始说起：

```cpp
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __Block_byref_a_0 *a; // by ref
    __Block_byref_b_1 *b; // by ref
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_a_0 *_a, __Block_byref_b_1 *_b, int flags=0) : a(_a->__forwarding), b(_b->__forwarding) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
```

在源码中我们可以看到：被 `__block` 修饰的局部变量 `__block int a`、`__block int b` 居然分别变成了 `__Block_byref_a_0`、`__Block_byref_b_1` 类型的结构体实例。

我们在 `__main_block_impl_0` 结构体中可以看到：**`__main_block_impl_0` 结构体实例持有指向`_Block_byref_a_0`、`__Block_byref_b_1` 结构体实例的指针。**

这里使用结构体指针 `a` 、结构体指针 `b` 说明 :`_Block_byref_a_0`、`__Block_byref_b_1` 类型的结构体并不在 `__main_block_impl_0` 结构体中，而**只是通过指针的形式引用，这是为了可以在多个不同的 Block 中使用 `__block` 修饰的变量。**

`__Block_byref_a_0`、`__Block_byref_b_1` 类型的结构体声明如下：

```cpp
struct __Block_byref_a_0 {
    void *__isa;
    __Block_byref_a_0 *__forwarding;
    int __flags;
    int __size;
    int a;
};

struct __Block_byref_b_1 {
    void *__isa;
    __Block_byref_b_1 *__forwarding;
    int __flags;
    int __size;
    int b;
};
```

拿第一个 `__Block_byref_a_0` 结构体定义来说明，`__Block_byref_a_0` 有 5 个部分：

1. `__isa`：标识对象类的 `isa` 实例变量
2. `__forwarding`：传入变量的地址
3. `__flags`：标志位
4. `__size`：结构体大小
5. `a`：存放实变量 `a` 实际的值。

再来看一下 `main()` 函数中，`__block int a`、`__block int b` 的赋值情况。

顺便把代码整理一下，使之简易一点：

```cpp
__Block_byref_a_0 a = {
    (void*)0,
    (__Block_byref_a_0 *)&a, 
    0, 
    sizeof(__Block_byref_a_0), 
    10
};

__Block_byref_b_1 b = {
    0,
    &b, 
    0, 
    sizeof(__Block_byref_b_1), 
    20
};
```

还是拿第一个`__Block_byref_a_0 a` 的赋值来说明。

可以看到 `__isa` 指针值传空，`__forwarding` 指向了局部变量 `a` 本身的地址，`__flags` 分配了 0，`__size` 为结构体的大小，`a` 赋值为 10。下图用来说明 `__forwarding` 指针的指向情况。

[![img](http://qncdn.bujige.net/images/iOS-Blocks-02-003.png)](http://qncdn.bujige.net/images/iOS-Blocks-02-003.png)

这下，我们知道 `__forwarding` 其实就是局部变量 `a` 本身的地址，那么我们就可以通过 `__forwarding` 指针来访问局部变量，同时也能对其进行修改了。

- 那`__forwarding` 既然这个指针指向自己本身的地址，那它存在的意义是什么呢？在下面“__block 变量存储域”部分会进行解释。

来看一下 Block 主体部分对应的 `__main_block_func_0` 结构体来验证一下。

```cpp
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    __Block_byref_a_0 *a = __cself->a; // bound by ref
    __Block_byref_b_1 *b = __cself->b; // bound by ref

    (a->__forwarding->a) = 20;
    (b->__forwarding->b) = 30;

    printf("a = %d, b = %d\n",(a->__forwarding->a), (b->__forwarding->b));
}
```

可以看到 `(a->__forwarding->a) = 20;` 和 `(b->__forwarding->b) = 30;` 是通过指针取值的方式来改变了局部变量的值。这也就解释了通过 `__block` 来修饰的变量，在 Block 的主体部分中改变值的原理其实是：通过**『指针传递』**的方式。

***

### __block修饰对象

__block可以指定任何类型的自动变量。下面来指定id类型的对象:

看一下__block变量的结构体：

```cpp
struct __Block_byref_obj_0 {
  void *__isa;
__Block_byref_obj_0 *__forwarding;
 int __flags;
 int __size;
 void (*__Block_byref_id_object_copy)(void*, void*);
 void (*__Block_byref_id_object_dispose)(void*);
 id obj;
};
```

被`__strong`修饰的`id`类型或对象类型自动变量的`copy`和`dispose`方法：

```cpp
static void __Block_byref_id_object_copy_131(void *dst, void *src) {
 _Block_object_assign((char*)dst + 40, *(void * *) ((char*)src + 40), 131);
}

static void __Block_byref_id_object_dispose_131(void *src) {
 _Block_object_dispose(*(void * *) ((char*)src + 40), 131);
}
```

同样，当Block持有被`__strong`修饰的`id`类型或对象类型自动变量时：

- 如果`__block`对象变量从栈复制到堆时，使用`_Block_object_assign`函数，
- 当堆上的`__block`对象变量被废弃时，使用`_Block_object_dispose`函数。

> 其中，_Block_object_assign相当于retain操作，将对象赋值在对象类型的结构体成员变量中。 _Block_object_dispose相当于release操作。  

这两个函数调用的时机是在什么时候呢？

| 函数                   | 被调用时机          |
| ---------------------- | ------------------- |
| __main_block_copy_0    | 从栈复制到堆时      |
| __main_block_dispose_0 | 堆上的Block被废弃时 |

 

```cpp
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_obj_0 *obj; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_obj_0 *_obj, int flags=0) : obj(_obj->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

可以看到，obj被添加到了__main_block_impl_0结构体中，它是__Block_byref_obj_0类型。

# Block 存储域

通过之前对 Block 本质的探索，我们知道了 Block 的本质是 Objective-C 对象。通过上述代码中 `impl.isa = &_NSConcreteStackBlock;`，可以知道该 Block 的类名为 `NSConcreteStackBlock`，根据名称可以看出，该 Block 是存于栈区中的。而与之相关的，还有 `_NSConcreteGlobalBlock`、`_NSConcreteMallocBlock`。

- _NSConcreteStackBlock.     //栈区
- _NSConcreteGlobalBlock   //数据区（全局区）
- _NSConcreteMallocBlock  // 堆区

![](http://ww1.sinaimg.cn/large/006y8mN6ly1g71h1y4borj30it0eh75z.jpg)

## 1. _NSConcreteGlobalBlock

在以下两种情况下使用 Block 的时候，Block 为 `NSConcreteGlobalBlock` 类对象。

1. 记述全局变量的地方，使用 Block 语法时；(这是因为在使用全局变量的地方不能使用自动变量，所以不存在对自动变量进行截获，由此 Block 用结构体的实例的内容不依赖于执行时的状态，所以整个程序中只需要一个实例)。
2. Block 语法的表达式中没有截获的自动变量时。

- `NSConcreteGlobalBlock` 类的 Block 存储在**『程序的数据区域』**。因为存放在程序的数据区域，所以即使在变量的作用域外，也可以通过指针安全的使用。

- 这种块不会捕捉任何状态（比如外围的变量等），运行时也无须有状态来参与。块所使用的整个内存区域。在编译期已经完全确定了，因此，全局块可以声明在全局内存里，而不需要在每次用到的时候于栈创建。

- 针对没有捕获自己主动变量的block来说，尽管用clang的rewrite-objc转化后的代码中仍显示`isa`指针指向`_NSConcretStackBlock`，可是实际上不是这样。它的类型还是全局的，具体可以参考一下这篇文章：[block存储区域——怎样验证block在栈上，还是堆上](https://blog.csdn.net/weixin_34216036/article/details/85689650)

- 记述全局变量的地方，使用 Block 语法示例代码：

```cpp
void (^myGlobalBlock)(void) = ^{
    printf("GlobalBlock\n");
};

int main() {
    myGlobalBlock();

    return 0;
}
```

通过对应 C++ 源码，我们可以发现：Block 结构体的成员变量 `isa` 赋值为：`impl.isa = &_NSConcreteGlobalBlock;`，说明该 Block 为 `NSConcreteGlobalBlock` 类对象。

## 2. _NSConcreteStackBlock

除了 **1. _NSConcreteGlobalBlock** 中提到的两种情形，其他情形下创建的 Block 都是 `NSConcreteStackBlock` 对象，平常接触的 Block 大多属于 `NSConcreteStackBlock` 对象。

`NSConcreteStackBlock` 类的 Block 存储在『栈区』的。如果其所属的变量作用域结束，则该 Block 就会被废弃。如果 Block 使用了 `__block` 变量，则当 `__block` 变量的作用域结束，则 `__block` 变量同样被废弃。

[![img](http://qncdn.bujige.net/images/iOS-Blocks-02-004.png)](http://qncdn.bujige.net/images/iOS-Blocks-02-004.png)

## 3. _NSConcreteMallocBlock

为了解决栈区上的 Block 在变量作用域结束被废弃这一问题，Block 提供了 **『复制』** 功能。可以将 Block 对象和 `__block` 变量从栈区复制到堆区上。当 Block 从栈区复制到堆区后，即使栈区上的变量作用域结束时，堆区上的 Block 和 `__block` 变量仍然可以继续存在，也可以继续使用。

[![img](http://qncdn.bujige.net/images/iOS-Blocks-02-005.png)](http://qncdn.bujige.net/images/iOS-Blocks-02-005.png)

此时，『堆区』上的 Block 为 `NSConcreteMallocBlock` 对象，Block 结构体的成员变量 isa 赋值为：`impl.isa = &_NSConcreteMallocBlock;`

那么，什么时候才会将 Block 从栈区复制到堆区呢？

这就涉及到了 Block 的自动拷贝和手动拷贝。

## Block 的自动拷贝和手动拷贝

### Block 的自动拷贝

在使用 ARC 时，大多数情形下编译器会自动进行判断，自动生成将 Block 从栈上复制到堆上的代码：

1. 将 Block 作为函数返回值返回时，会自动拷贝；
2. 向方法或函数的参数中传递 Block 时，使用以下两种方法的情况下，会进行自动拷贝，否则就需要手动拷贝：
   1. Cocoa 框架的方法且方法名中含有 `usingBlock` 等时；
   2. `Grand Central Dispatch（GCD）` 的 API。

3. 将 Block 赋值给类的附有 `__strong`修饰符的`id`类型或 Block 类型成员变量时

### Block 的手动拷贝

我们可以通过『copy 实例方法（即 `alloc / new / copy / mutableCopy`）』来对 Block 进行手动拷贝。当我们不确定 Block 是否会被遗弃，需不需要拷贝的时候，直接使用 copy 实例方法即可，不会引起任何的问题。

关于 Block 不同类的拷贝效果总结如下：

|        Block 类        |    存储区域    |   拷贝效果   |
| :--------------------: | :------------: | :----------: |
| _NSConcreteStackBlock  |      栈区      | 从栈拷贝到堆 |
| _NSConcreteGlobalBlock | 程序的数据区域 |   不做改变   |
| _NSConcreteMallocBlock |      堆区      | 引用计数增加 |

# __block 变量存储域
## __block 变量的拷贝

在使用 `__block` 变量的 Block 从栈复制到堆上时，`__block` 变量也会受到如下影响：

| __block 变量的配置存储区域 |   Block 从栈复制到堆时的影响    |
| :------------------------: | :-----------------------------: |
|            栈区            | 从栈复制到堆，并被 Block 所持有 |
|            堆区            |         被 Block 所持有         |

当然，如果不再有 Block 引用该 `__block` 变量，那么 `__block` 变量也会被废除。和 ARC 规则一样。具体过程看下面几张图就可以理解。

1. ![](http://ww3.sinaimg.cn/large/006y8mN6ly1g71gh1y2dzj30gq0ad0ub.jpg)
2. ![](http://ww1.sinaimg.cn/large/006y8mN6ly1g71gi05bk9j30mi0j90wh.jpg)
3. ![](http://ww1.sinaimg.cn/large/006y8mN6ly1g71giie4tej30ou0jltc6.jpg)

- 如果我们理解了上面这部分内容，我们也就知道了上面遗留问题（为什么`__block 变量`用结构体成员变量`__forwarding`）的答案：

  “不管 `__block` 变量配置在栈上还是堆上，都能够正确地访问该变量”。正如这句话所述，通过 Block 的复制，`__block` 变量也从栈复制到堆。此时可同时访问栈上的`__block` 变量和堆上的`__block` 变量。

  ![](http://ww4.sinaimg.cn/large/006y8mN6ly1g71gdh7x4vj30pf0ju778.jpg)

在这里举个例子，假设`val`是一个被`__block 说明符`修饰的整型变量：

 ```cpp
^{++val;}
 ```

然后在Block语法之后使用与 Block 无关的变量。

```cpp
++val;
```

以上两种源代码均可转换为如下形式：

```cpp
++(val.__forwarding->val);
```

在变换 Block 语法的函数中，该变量 val 为复制到堆上的 `__block` 变量结构体实例，而使用的与 Block 无关的变量 val，为复制前栈上的 `__block`变量结构体实例。

但是栈上的 `__block`变量结构体实例在`__block`变量从栈复制到堆上时，会将成员变量 `__forwarding`到值替换成复制目标堆上的`__block`变量结构体实例的地址。如上面那张图所示。

 # block 循环引用

举个例子，下面是一个类的`init`方法,`blk_`是该类的一个成员变量：

```objectivec
- (id)init {
	self = [super init];
	blk_ = ^{NSLog(@"self = %@", self);};
	return self;
}
```

初始化这个类的实例时，就会造成循环引用，因为 Block 语法赋值在了成员变量 `blk_`中，因此通过 Block 语法生成在栈上的 Block 此时由栈复制到堆，并持有所使用的 self。self 持有 Block，Block 持有 self。这正是循环引用。

注意：**Block 内使用类的成员变量实际截获的是这个类本身（self）。**对编译器来说，成员变量只不过是对象结构体的成员变量。所以如何Block是该类的成员变量，截获该类其他成员变量时，会造成循环引用。