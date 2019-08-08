[TOC]

# 前言

- 这部分内容我是拜读《iOS高级编程》过程中总结而成的，个人觉得这本书的作者确实很牛，在他当时苹果NSObject类的源码还没有公布时就根据Cocoa框架的互换框架GNUstep对苹果的实现推测的不离大谱。但就可惜就是出版时间太久了，其中很多源代码实现目前都改了，并且苹果已经公开了这部分的源代码，所以我建议学习这本书时着重于对作者内存管理方式、概念和思维的理解，实现部分留个印象再去剖源码。

***

# 准备工作
- 了解计算机内存存储区域：[iOS-MRC与ARC区别以及五大内存区](https://blog.csdn.net/qq_42347755/article/details/96974898)

- 项目打开MRC的方式：

  ​	![项目打开MRC的方式](https://img-blog.csdn.net/20160522024533316?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

***

# 内存管理（引用计数）的理解
## 对象操作与OC中方法的对应

| 对象操作       | OC中方法                     |
| -------------- | ---------------------------- |
| 生成并持有对象 | alloc/new/copy/mutableCopy等 |
| 持有对象       | retain                       |
| 释放对象       | release                      |
| 废弃对象       | dealloc                      |

- 这些OC内存管理的方法，实际上不包括在该语言中，而是包含在Cocoa框架中用于 OS X、iOS应用开发，Cocoa框架中的Foundation框架类库的NSObject类担负内存管理的职责

## 内存管理的思考方式：

- 下面会反复提到持有的概念，网上普遍说法持有就是强引用，我说说我理解的持有：**表示持有者还需使用这个对象，并要负责这个对象的释放。**

1. 自己生成的对象，自己持有。
   - 使用以下名称开头的方法名意味着自己生成的对象只有自己持有：
     - alloc
     - new
     - copy
     - mutableCopy
   - 	至于copy / mutableCopy 为什么也会被纳入这个表单可见[详解iOS开发中复制对象](https://blog.csdn.net/qq_42347755/article/details/89057135)
   - 	[NSObject new] 与 [[NSObject alloc] init] 完全等价
   - 	持有的本质其实就是强引用
   
2. 非自己生成的对象，自己也能持有。

   - 用 alloc / new / copy / mutableCopy 以外的方法取得的对象，因为非自己生成并持有，所以自己不是该对象的持有者。（比如 NSMutableArray 类的 array方法）

     ```objective-c
     //示例代码展示如何实现取得的对象存在，但自己不持有对象
     - (id)object {
     	id object = [[NSObject alloc] init];//自己持有对象
     	[obj autorelease]//取得对象存在，但自己不持有对象
     	return obj;
     }
     ```

     

   - 数组变量通过 NSMutableArray 类的 array 方法初始化拿到了生成对象存储地址的指针，但还需使用 retain 方法才可以持有对象。

   - 至于“非自己生成的对象，自己也能持有”的原理，下面讲到的 autorelease 方法会告诉你答案。

3. 不再需要自己持有的对象时释放。

   - 释放时使用release方法。

4. 非自己持有的对象无法释放

   - 释放非自己持有的对象会导致程序崩溃。
   
## 神奇的打印实验

- 我们看个🌰：

```objective-c
NSString *str = @"abcdefg"; 
NSLog(@"Str = %lu",(unsigned long)[str retainCount]); 

NSNumber *num = @11; 
NSLog(@"Str = %lu",(unsigned long)[str retainCount]); 

NSNumber *longNum = @222222222222222222;
NSLog(@"longNum = %lu",(unsigned long)[longNum retainCount]);

NSNumber *floatNum = @3.14;
NSLog(@"floatNum = %f",(unsigned long)[floatNum retainCount]);

NSMutableString *str1 = [[NSMutableString alloc]initWithString:@"abcdefg"];
NSLog(@"Str1 = %lu",(unsigned long)[str1 retainCount]); 
```

结果是：

```objective-c
Str     = 18446744073709551615
num     = 9223372036854775807
floatNum = 1
longNum = 1
Str1    = 1
```

- 刚看到这个例子我也很困惑，为什么刚初始化创建的 NSString 对象和 NSNumber 对象引用计数这么大，不过查阅资料后解决了这个问题

- 在**Effective Objective 2.0**的第36条这样解释道：

> 对象的引用计数非常大是因为，这些对象都是“单例对象”，所以其引用计数都很大。系统会尽可能把NSString实现成单例对象。如果字符串像例子上出现的情况这样，是个编译器常量（compile-time constant）,那么就可以这样来实现了。在这种情况下，编译器会把NSString对象所表示的数据放到应用程序的二进制文件里，这样的话，运行程序时就可以直接使用了，无需再创建NSString对象。NSNumber也类似，他使用了一种叫做“标签指针”（tagged pointer）的概念来标注特定类型的数值。这种做法不适用NSNumber对象，而是把与数值有关的全部消息都放在指针值里面。运行时系统会在消息派发期间检测这种标签指针，并对它执行相应操作，使其行为看上去和真正的NSNumber对象一样。这种优化只在某些场合使用，比如范例中的浮点数对象就没有优化，所以其保留计数就是1。并且这种单例对象的引用计数也不会改变。

- NSString类有点特殊，具体可以看这篇博客 [iOS-MRC与ARC区别以及五大内存区](https://blog.csdn.net/qq_42347755/article/details/96974898)

#autorelease

- 在这部分，我开始是打算按《iOS高级编程》写【alloc】【retain】【release】【dealloc】的实现的，但在对苹果最新源码初步剖析以后，发现GNUstep的实现与苹果真正自己的实现还是有一定差距的，这里不再赘述，等在写剖析runtime源码博客的时候再说实现部分。在这只需：

  1. 知道 alloc 方法生成对象并为其分配内存，引用计数变为1；
  2. retain 方法会使调用者持有对象，对象的引用计数+1；
  3. release 方法则会解除引用且使对象引用计数-1；
  4. dealloc 方法销毁对象，对象所占内存解除分配，并放回“可用内存池中”；

- 首先说说什么是自动释放池：自动释放池是OC的一种内存自动回收机制，可以将一些临时变量通过自动释放池来回收统一释放。自动释放池本事销毁的时候，池子里面所有的对象都会做一次release操作 　　

- 调用 autorelease 方法，就会把该对象放到离自己最近的自动释放池中（栈顶的释放池，多重自动释放池嵌套是以栈的形式存取的），即：使对象的持有权转移给了自动释放池（即注册到了自动释放池中），从而解释了上面内存管理思考方式第二条的疑问：调用方拿到了对象，但这个对象还不被调用方所持有。

  ![](http://ww3.sinaimg.cn/large/006tNc79ly1g5asqxgh0gj30sh0ljn3w.jpg)

- 注意：autorelease 方法不会改变调用者的引用计数，它只是改变了对象释放时机，不再让程序员负责释放这个对象，而是交给自动释放池去处理。

- 之前说过了，autorelease 方法相当于把调用者注册到 autoreleasepool 中，ARC环境下不能显式地调用 autorelease 方法和显式地创建 NSAutoreleasePool 对象，但可以使用`@autoreleasepool { }`块代替（并不代表块中所有内容都被注册到了自动释放池中）

  ![](http://ww4.sinaimg.cn/large/006tNc79ly1g5epf5m91zj30kd097gnr.jpg)

  - 所以使用自动释放池的时机很重要，什么时候使用自动释放池呢，官方文档中有两种情况：

    ​	1. **If you write a loop that creates many temporary object**. 循环中创建了许多临时对象，在循环里使用自动释放池，用来减少高内存占用。

    ​	2. **If you spawn a secondary thread.** 开启子线程的时候要自己创建自己的释放池，否则可能会发生内存泄漏。

  - 在这说一下第一条，如果for循环中创建了大量的临时对象，如下例

    ```objective-c
    for (int i = 0; i < MAX; i++){//这里的MAX很大
    	UIImage *image = [UIImage new];
    	...
    	//大量产生autorelease的对象
    	//由于没有废弃NASAutoreleasePool对象
    	//最终导致内存不足
    } 
    
    ```

    在此之前我一直以为一次循环介绍后生成的临时对象就会被释放，查阅资料后发现我错了，这时我们手动生成一个autoreleasepool，就能防止内存上涨的情况发生。

    ```objective-c
    for (int i = 0; i < MAX; i++){//这里的MAX很大
    	@autoreleasepool {
    		UIImage *image = [UIImage new];
    	...
    	}
    } 
    ```

    - 一个runloop对应一个autoreleasepool

      ![](http://ww2.sinaimg.cn/large/006tNc79ly1g5eq3r6ogjj30hq06egmx.jpg)

# 一些细节

- 对象所占的内存在“解除分配”（deallocated）之后，只是放回“可用内存池”（avaiable pool）。如果执行 NSLog 时尚未覆写对象内存，那么该对象依然有效，这时程序不会崩溃。
- NSZone（区域）是防止内存碎片化而引入的结构，但现在但运行时系统本身已极具效率，使用区域管理内存反而会引起内存使用效率低下及代码复杂化等问题，故目前已被忽略。