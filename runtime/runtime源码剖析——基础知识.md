[TOC]

> **本文作者：** 行走少年郎
>
> **本文链接：** https://www.bujige.net/blog/iOS-Runtime-01.html

# 1. 什么是 Runtime？

我们都知道，将源代码转换为可执行的程序，通常要经过三个步骤：**编译**、**链接**、**运行**。不同的编译语言，在这三个步骤中所进行的操作又有些不同。

`C 语言` 作为一门静态类语言，在编译阶段就已经确定了所有变量的数据类型，同时也确定好了要调用的函数，以及函数的实现。

而 `Objective-C 语言` 是一门动态语言。在编译阶段并不知道变量的具体数据类型，也不知道所真正调用的哪个函数。只有在运行时间才检查变量的数据类型，同时在运行时才会根据函数名查找要调用的具体函数。这样在程序没运行的时候，我们并不知道调用一个方法具体会发生什么。

`Objective-C 语言` 把一些决定性的工作从编译阶段、链接阶段推迟到 **运行时阶段** 的机制，使得 `Objective-C` 变得更加灵活。我们甚至可以在程序运行的时候，动态的去修改一个方法的实现，这也为大为流行的『热更新』提供了可能性。

而实现 `Objective-C 语言` **运行时机制** 的一切基础就是 `Runtime`。

`Runtime` 实际上是一个库，这个库使我们可以在程序运行时动态的创建对象、检查对象，修改类和对象的方法。

# 2. 消息机制的基本原理

`Objective-C 语言` 中，对象方法调用都是类似 `[receiver selector];` 的形式，其本质就是让对象在运行时发送消息的过程。

我们来看看方法调用 `[receiver selector];` 在『编译阶段』和『运行阶段』分别做了什么？

1. 编译阶段：`[receiver selector];`方法被编译器转换为:
   1. `objc_msgSend(receiver，selector)` （不带参数）
   2. `objc_msgSend(recevier，selector，org1，org2，…)`（带参数）

2. 运行时阶段：消息接受者`recever`寻找对应的`selector`。

   1. 通过 `recevier` 的 `isa 指针` 找到 `recevier` 的 `Class（类）`；
   2. 在 `Class（类）` 的`cache`中找这个`selector`，找不到的话在 `bits`其中的 `class_rw_t `  中找到 `methods` 对应的 `selector`；
   3. 如果在 `Class（类）` 中没有找到这个 `selector`，就继续在它的 `superClass（父类）`中寻找；
   4. 一旦找到对应的 `selector`，直接执行 `recever` 对应 `selector` 方法实现的 `IMP（方法实现）`。
   5. 若找不到对应的 `selector`，消息被转发或者临时向 `recever` 添加这个 `selector` 对应的实现方法，否则就会发生崩溃。

在上述过程中涉及了好几个新的概念：`objc_msgSend`、`isa 指针`、`Class（类）`、`IMP（方法实现）` 等，下面我们来具体讲解一下各个概念的含义。

------

# 3. Runtime 中的概念解析

> 这里原笔者讲解的Class类源码版本太老了，我删减掉了，最新版可以我的参考这篇文章:[runtime源码剖析—— isa 是什么](https://blog.csdn.net/qq_42347755/article/details/98786218)

## 3.6 方法（Method）

`object_class 结构体` 的 `methodLists（方法列表）`中存放的元素就是 `方法（Method）`。

先来看下 `objc/runtime.h` 中，表示 `方法（Method）` 的 `objc_method 结构体` 的数据结构：

```cpp
/// An opaque type that represents a method in a class definition.
/// 代表类定义中一个方法的不透明类型
typedef struct objc_method *Method;

struct objc_method {
    SEL _Nonnull method_name;                    // 方法名
    char * _Nullable method_types;               // 方法类型
    IMP _Nonnull method_imp;                     // 方法实现
};
```

可以看到，`objc_method 结构体` 中包含了 `方法名（method_name）`，`方法类型（method_types）` 和 `方法实现（method_imp）`。下面，我们来了解下这三个变量。

1. `SEL method_name; // 方法名`

```cpp
/// An opaque type that represents a method selector.
typedef struct objc_selector *SEL;
```

`SEL` 是一个指向 `objc_selector 结构体` 的指针，但是在 runtime 相关头文件中并没有找到明确的定义。不过，通过测试我们可以得出： `SEL` 只是一个保存方法名的字符串。

```objectivec
SEL sel = @selector(viewDidLoad);
NSLog(@"%s", sel);              // 输出：viewDidLoad
SEL sel1 = @selector(test);
NSLog(@"%s", sel1);             // 输出：test
```

1. `IMP method_imp; // 方法实现`

```objectivec
/// A pointer to the function of a method implementation. 
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ ); 
#else
typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...); 
#endif
```

`IMP` 的实质是一个函数指针，所指向的就是方法的实现。`IMP`用来找到函数地址，然后执行函数。

1. `char * method_types; // 方法类型`

方法类型 `method_types` 是个字符串，用来存储方法的参数类型和返回值类型。

> 到这里， `Method` 的结构就已经很清楚了，`Method` 将 `SEL（方法名）` 和 `IMP（函数指针）` 关联起来，当对一个对象发送消息时，会通过给出的 `SEL（方法名）` 去找到 `IMP（函数指针）` ，然后执行。

------

# 4. Runtime 消息转发

在 **2. 消息机制的基本原理** 最后一步中我们提到：若找不到对应的 `selector`，消息被转发或者临时向 `receiver` 添加这个 `selector` 对应的实现方法，否则就会发生崩溃。

> 当一个方法找不到的时候，Runtime 提供了 **消息动态解析**、**消息接受者重定向**、**消息重定向** 等三步处理消息，具体流程如下图所示：

[![img](http://qncdn.bujige.net/images/iOS-Runtime-01-003.png)](http://qncdn.bujige.net/images/iOS-Runtime-01-003.png)

## 4.1 消息动态解析

Objective-C 运行时会调用 `+resolveInstanceMethod:` 或者 `+resolveClassMethod:`，让你有机会提供一个函数实现。我们可以通过重写这两个方法，添加其他函数实现，并返回 `YES`， 那运行时系统就会重新启动一次消息发送的过程。

主要用的的方法如下：

```objectivec
// 类方法未找到时调起，可以在此添加类方法实现
+ (BOOL)resolveClassMethod:(SEL)sel;
// 对象方法未找到时调起，可以在此对象方法实现
+ (BOOL)resolveInstanceMethod:(SEL)sel;

/** 
 * class_addMethod    向具有给定名称和实现的类中添加新方法
 * @param cls         被添加方法的类
 * @param name        selector 方法名
 * @param imp         实现方法的函数指针
 * @param types imp   指向函数的返回值与参数类型
 * @return            如果添加方法成功返回 YES，否则返回 NO
 */
BOOL class_addMethod(Class cls, SEL name, IMP imp, 
                const char * _Nullable types);
```

举个例子：

```objectivec
#import "ViewController.h"
#include "objc/runtime.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 执行 fun 函数
    [self performSelector:@selector(fun)];
}

// 重写 resolveInstanceMethod: 添加对象方法实现
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(fun)) { // 如果是执行 fun 函数，就动态解析，指定新的 IMP
        class_addMethod([self class], sel, (IMP)funMethod, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}

void funMethod(id obj, SEL _cmd) {
    NSLog(@"funMethod"); //新的 fun 函数
}

@end
```

> 打印结果：
> 2019-06-12 10:25:39.848260+0800 runtime[14884:7977579] funMethod

从上边的例子中，我们可以看出，虽然我们没有实现 `fun` 方法，但是通过重写 `resolveInstanceMethod:` ，利用 `class_addMethod` 方法添加对象方法实现 `funMethod` 方法，并执行。从打印结果来看，成功调起了`funMethod` 方法。

> 我们注意到 class_addMethod 方法中的特殊参数 `v@:`，具体可参考官方文档中关于 `Type Encodings` 的说明：[传送门](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100)

## 4.2 消息接受者重定向

如果上一步中 `+resolveInstanceMethod:` 或者 `+resolveClassMethod:` 没有添加其他函数实现，运行时就会进行下一步：消息接受者重定向。

如果当前对象实现了 `-forwardingTargetForSelector:`，Runtime 就会调用这个方法，允许我们将消息的接受者转发给其他对象。

用到的方法：

```objectivec
// 重定向方法的消息接收者，返回一个类或实例对象
- (id)forwardingTargetForSelector:(SEL)aSelector;
```

> 注意：这里`+resolveInstanceMethod:` 或者 `+resolveClassMethod:`无论是返回 `YES`，还是返回 `NO`，只要其中没有添加其他函数实现，运行时都会进行下一步。

举个例子：

```objectivec
#import "ViewController.h"
#include "objc/runtime.h"

@interface Person : NSObject

- (void)fun;

@end

@implementation Person

- (void)fun {
    NSLog(@"fun");
}

@end

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 执行 fun 方法
    [self performSelector:@selector(fun)];
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    return YES; // 为了进行下一步 消息接受者重定向
}

// 消息接受者重定向
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(fun)) {
        return [[Person alloc] init];
        // 返回 Person 对象，让 Person 对象接收这个消息
    }
    
    return [super forwardingTargetForSelector:aSelector];
}
```

> 打印结果：
> 2019-06-12 17:34:05.027800+0800 runtime[19495:8232512] fun

可以看到，虽然当前 `ViewController` 没有实现 `fun` 方法，`+resolveInstanceMethod:` 也没有添加其他函数实现。但是我们通过 `forwardingTargetForSelector` 把当前 `ViewController` 的方法转发给了 `Person` 对象去执行了。打印结果也证明我们成功实现了转发。

我们通过 `forwardingTargetForSelector` 可以修改消息的接收者，该方法返回参数是一个对象，如果这个对象是不是 `nil`，也不是 `self`，系统会将运行的消息转发给这个对象执行。否则，继续进行下一步：消息重定向流程。

## 4.3 消息重定向

如果经过消息动态解析、消息接受者重定向，Runtime 系统还是找不到相应的方法实现而无法响应消息，Runtime 系统会利用 `-methodSignatureForSelector:` 方法获取函数的参数和返回值类型。

- 如果`-methodSignatureForSelector:`返回了一个`NSMethodSignature`对象（函数签名），Runtime 系统就会创建一个`NSInvocation`对象，并通过`-forwardInvocation:`消息通知当前对象，给予此次消息发送最后一次寻找 IMP 的机会。

  - 如果 `-methodSignatureForSelector:` 返回 `nil`。则 Runtime 系统会发出 `-doesNotRecognizeSelector:` 消息，程序也就崩溃了。

所以我们可以在 `-forwardInvocation:` 方法中对消息进行转发。

用到的方法：

```objectivec
// 获取函数的参数和返回值类型，返回签名
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;

// 消息重定向
- (void)forwardInvocation:(NSInvocation *)anInvocation；
```



举个例子：

```objectivec
#import "ViewController.h"
#include "objc/runtime.h"

@interface Person : NSObject

- (void)fun;

@end

@implementation Person

- (void)fun {
    NSLog(@"fun");
}

@end

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 执行 fun 函数
    [self performSelector:@selector(fun)];
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    return YES; // 为了进行下一步 消息接受者重定向
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    return nil; // 为了进行下一步 消息重定向
}

// 获取函数的参数和返回值类型，返回签名
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    if ([NSStringFromSelector(aSelector) isEqualToString:@"fun"]) {
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    
    return [super methodSignatureForSelector:aSelector];
}

// 消息重定向
- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL sel = anInvocation.selector;   // 从 anInvocation 中获取消息
    
    Person *p = [[Person alloc] init];

    if([p respondsToSelector:sel]) {   // 判断 Person 对象方法是否可以响应 sel
        [anInvocation invokeWithTarget:p];  // 若可以响应，则将消息转发给其他对象处理
    } else {
        [self doesNotRecognizeSelector:sel];  // 若仍然无法响应，则报错：找不到响应方法
    }
}
@end
```

> 打印结果：
> 2019-06-13 13:23:06.935624+0800 runtime[30032:8724248] fun

可以看到，我们在 `-forwardInvocation:` 方法里面让 `Person` 对象去执行了 `fun` 函数。

既然 `-forwardingTargetForSelector:` 和 `-forwardInvocation:` 都可以将消息转发给其他对象处理，那么两者的区别在哪？

区别就在于 `-forwardingTargetForSelector:` 只能将消息转发给一个对象。而 `-forwardInvocation:` 可以将消息转发给多个对象。

以上就是 Runtime 消息转发的整个流程。

结合之前讲的 **2. 消息机制的基本原理**，就构成了整个消息发送以及转发的流程。下面我们来总结一下整个流程。

------

# 5. 消息发送以及转发机制总结

调用 `[receiver selector];` 后，进行的流程：

1. 编译阶段：`[receiver selector];`方法被编译器转换为:
   1. `objc_msgSend(receiver，selector)` （不带参数）
   2. `objc_msgSend(recevier，selector，org1，org2，…)`（带参数）
2. 运行时阶段：消息接受者`recever`寻找对应的`selector`。

   1. 通过 `recevier` 的 `isa 指针` 找到 `recevier` 的 `class（类）`；
   2. 在 `class（类）` 的 `method list（方法列表）` 中找对应的 `selector`；
   3. 如果在 `class（类）` 中没有找到这个 `selector`，就继续在它的 `superclass（父类）`中寻找；
   4. 一旦找到对应的 `selector`，直接执行 `recever` 对应 `selector` 方法实现的 `IMP（方法实现）`。
   5. 若找不到对应的 `selector`，Runtime 系统进入消息转发机制。

3. 运行时消息转发阶段：

   1. 动态解析：通过重写 `+resolveInstanceMethod:` 或者 `+resolveClassMethod:`方法，利用 `class_addMethod`方法添加其他函数实现；

   2. 消息接受者重定向：如果上一步添加其他函数实现，可在当前对象中利用 `-forwardingTargetForSelector:` 方法将消息的接受者转发给其他对象；

   3. 消息重定向：如果上一步没有返回值为`nil`，则利用`-methodSignatureForSelector:`方法获取函数的参数和返回值类型。

      1. 如果`-methodSignatureForSelector:`返回了一个`NSMethodSignature`对象（函数签名），Runtime 系统就会创建一个`NSInvocation`对象，并通过`-forwardInvocation:`消息通知当前对象，给予此次消息发送最后一次寻找 IMP 的机会。

         1. 如果 `-methodSignatureForSelector:` 返回 `nil`。则 Runtime 系统会发出 `-doesNotRecognizeSelector:` 消息，程序也就崩溃了。