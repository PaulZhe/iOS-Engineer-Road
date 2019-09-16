[TOC]

# 多线程编程

- 在说GCD之前我们先来了解一下多线程，多线程编程相信读到这篇博客的大家在写代码时多多少少都会遇到过，这里就不再赘述这个概念了，我们来看看多线程编程的优缺点。

- 在上次我在[从atomic关键字说到多线程安全（内含iOS给代码加锁方法总结)](https://blog.csdn.net/qq_42347755/article/details/97753122)中讲到了线程安全的问题，不过着重只是讲了多线程编程中数据竞争这一个类型的问题，通过下图我们可以直观的了解到多线程编程的其他问题：

  ![](http://ww3.sinaimg.cn/large/006tNc79ly1g5t5qp1vglj30xl0erdme.jpg)

- APP在启动时，通过最先执行的线程，即“主线程”来描绘用户界面，处理触摸屏幕的事件等。如果在该主线程中进行长时间的处理，就会妨碍主线程的执行（阻塞）。会妨碍主线程中被称为RunLoop的主循环的执行，从而导致不能更新用户界面、应用程序的画面长时间停滞等问题。这也就是多线程编程的优点：使用多线程编程，在执行长时间的处理时仍可以保证用户界面的响应性能。

  ![](http://ww3.sinaimg.cn/large/006tNc79ly1g5t5x8ouw3j30ig0qtwmx.jpg)



# GCD概要

- 下来我们说回 GCD 。什么是GCD？以下摘自苹果的官方说明：

- Grand Central Dispatch（GCD）是异步执行任务的技术之一。一般将应用程序中记述的线程管理用的代码在系统级中实现。开发者只需要定义想执行的任务并追加到适当的 Dispatch Queue 中，GCD 就能生成必要的线程并执行任务。由于线程管理是作为系统的一部分来实现的，因此可统一管理，也可执行任务，这样就比以前的线程更有效率。

- GCD大大简化了偏于复杂的多线程编程的源代码。



# GCD任务和队列

学习 GCD 之前，先来了解 GCD 中两个核心概念：**『任务』** 和 **『队列』**。

- 任务：就是你在线程中执行的那段代码。在 GCD 中是放在 block 中的。执行任务有两种方式：**『同步执行』** 和 **『异步执行』**。两者的主要区别是：**是否等待队列的任务执行结束，以及是否具备开启新线程的能力。**

  - 同步执行（sync）：

    - 同步添加任务到指定的队列中，在添加的任务执行结束之前，会一直等待，直到队列里面的任务完成之后再继续执行。
    - 只能在当前线程中执行任务，不具备开启新线程的能力。

  - 异步执行（async）：

    - 异步添加任务到指定的队列中，它不会做任何等待，可以继续执行任务。
- 可以在新的线程中执行任务，具备开启新线程的能力。

>注意：异步执行（async）虽然具有开启新线程的能力，但是并不一定开启新线程。这跟任务所指定的队列类型有关（下面会讲）。
>

**注意**：***这里的任务是以块的形式给出的，要将整个块中的内容理解为一个任务，再将其添加到相应的任务队列中，这对后面理解队列嵌套很有帮助。***

- 队列：在下文GCD的API中会详细分析 `Dispatch Queue`，这里只作简单介绍：在 GCD 中有两种队列：**『串行队列』** 和 **『并发队列』**。两者都符合 FIFO（先进先出）的原则。两者的主要区别是：**执行顺序不同，以及开启线程数不同。**

  - 串行队列（Serial Dispatch Queue）：

    - 每次只有一个任务被执行。让任务一个接着一个地执行。（只开启一个线程，一个任务执行完毕后，再执行下一个任务）

  - 并发队列（Concurrent Dispatch Queue）：

    - 可以让多个任务并发（同时）执行。（可以开启多个线程，并且同时执行任务）

    - **这个队列中的任务也是按照FIFO(先来后到的顺序)开始执行，注意是开始，但是它们的执行结束时间是不确定的，取决于每个任务的耗时。**
    
      > 注意：**并发队列** 的并发功能只有在异步（dispatch_async）方法下才有效。
      >

# GCD 的使用步骤

1. 创建一个队列（串行队列或并发队列）；

2. 将任务追加到任务的等待队列中，然后系统就会根据任务类型执行任务（同步执行或异步执行）。

## 队列的创建方法
- 下文GCD的API中介绍 `Dispatch Queue`时会讲。
## 任务的创建方法

GCD 提供了同步执行任务的创建方法 `dispatch_sync` 和异步执行任务创建方法 `dispatch_async`。

```objective-c
// 同步执行任务创建方法
dispatch_sync(queue, ^{
    // 这里放同步执行任务代码
});
// 异步执行任务创建方法
dispatch_async(queue, ^{
    // 这里放异步执行任务代码
});
```

- `dispatch_async`函数的`async`意味着“非同步”，将指定的Block非同步地追加到指定的Dispatch Queue中，**不用等待Block处理执行结束**。

- `dispatch_sync`函数的`sync`意味着“同步”，将指定的Block同步地追加到指定的Dispatch Queue中，在追加Block执行结束之前，`dispatch_sync`函数会一直等待。“等待”意味着当前线程停止。

## 任务和队列不同组合方式的区别

我们先来考虑最基本的使用，也就是当前线程为 **『主线程』** 的环境下，**『不同队列』+『不同任务』** 简单组合使用的不同区别。暂时不考虑 **『队列中嵌套队列』** 的这种复杂情况。

**『主线程』**中，**『不同队列』**+**『不同任务』**简单组合的区别：

|     区别      |           并发队列           |             串行队列              |            主队列            |
| :-----------: | :--------------------------: | :-------------------------------: | :--------------------------: |
| 同步（sync）  | 没有开启新线程，串行执行任务 |   没有开启新线程，串行执行任务    |        死锁卡住不执行        |
| 异步（async） |  有开启新线程，并发执行任务  | 有开启新线程（1条），串行执行任务 | 没有开启新线程，串行执行任务 |

> 注意：从上边可看出： **『主线程』** 中调用 **『主队列』+『同步执行』** 会导致死锁问题。
> 这是因为 **主队列中追加的同步任务** 和 **主线程本身的任务** 两者之间相互等待，阻塞了 **『主队列』**，最终造成了主队列所在的线程（主线程）死锁问题。
> 而如果我们在 **『其他线程』** 调用 **『主队列』+『同步执行』**，则不会阻塞 **『主队列』**，自然也不会造成死锁问题。最终的结果是：**不会开启新线程，串行执行任务**。

## 队列嵌套情况下，不同组合方式区别

除了上边提到的『主线程』中调用『主队列』+『同步执行』会导致死锁问题。实际在使用『串行队列』的时候，也可能出现阻塞『串行队列』所在线程的情况发生，从而造成死锁问题。这种情况多见于同一个串行队列的嵌套使用。

比如下面代码这样：在『异步执行』+『串行队列』的任务中，又嵌套了『当前的串行队列』，然后进行『同步执行』。

```objective-c
dispatch_queue_t queue = dispatch_queue_create("test.queue", DISPATCH_QUEUE_SERIAL);
dispatch_async(queue, ^{    // 异步执行 + 串行队列
    dispatch_sync(queue, ^{  // 同步执行 + 当前串行队列
        // 追加任务 1
        [NSThread sleepForTimeInterval:2];              // 模拟耗时操作
        NSLog(@"1---%@",[NSThread currentThread]);      // 打印当前线程
    });
});
```

> 执行上面的代码会导致 **串行队列中追加的任务** 和 **串行队列中原有的任务** 两者之间相互等待，阻塞了『串行队列』，最终造成了串行队列所在的线程（子线程）死锁问题。



- 关于 『队列中嵌套队列』这种复杂情况，这里也简单做一个总结。不过这里只考虑同一个队列的嵌套情况，关于多个队列的相互嵌套情况还请自行研究

**『不同队列』+『不同任务』** 组合，以及 **『队列中嵌套队列』** 使用的区别：

|     区别      | 『异步执行+并发队列』嵌套『同一个并发队列』 | 『同步执行+并发队列』嵌套『同一个并发队列』 | 『异步执行+串行队列』嵌套『同一个串行队列』 | 『同步执行+串行队列』嵌套『同一个串行队列』 |
| :-----------: | :-----------------------------------------: | :-----------------------------------------: | :-----------------------------------------: | :-----------------------------------------: |
| 同步（sync）  |       没有开启新的线程，串行执行任务        |        没有开启新线程，串行执行任务         |               死锁卡住不执行                |               死锁卡住不执行                |
| 异步（async） |         有开启新线程，并发执行任务          |         有开启新线程，并发执行任务          |     有开启新线程（1 条），串行执行任务      |     有开启新线程（1 条），串行执行任务      |

#GCD的API

## Dispatch Queue

- 首先回顾一下苹果官方对 GCD 的说明：**开发者要做的只是定义想执行的任务并追加到合适的 Dispatch Queue 中**

  这句话用源代码表示如下：

  ```objective-c
  dispatch_async(queue, ^{
  	/*
  	* 想执行的任务
  	*/
  });
  ```

- "Dispatch Queue" 就是执行处理的等待队列。按照追加的顺序（先进先出 FIFO，First-In-First-Out）执行处理。

  ![](http://ww2.sinaimg.cn/large/006tNc79ly1g5t92zg2jwj30y90eednq.jpg)

- 另外在执行处理时存在两种 Dispatch Queue :
  - Serial Dispatch ：一种是等待现在执行中的任务处理结束。
  - Concurrent Dispatch Queue:不等待现在执行中的任务处理结束。

![](http://ww1.sinaimg.cn/large/006tNc79ly1g5t9aej9w6j30xd0n7n7s.jpg)

![](http://ww1.sinaimg.cn/large/006tNc79ly1g5t9bql1x3j30ye0kiqd8.jpg)

- Concurrent Dispatch Queue并行执行的处理数：Concurrent Dispatch Queue不用等待任务处理结束，可**并行执行**多个处理，**并行执行**的处理数由CPU核数以及CPU负荷等当前系统的状态决定。所谓 “**并行执行**” ，就是使用多个线程同时执行多个处理。
- iOS和OS X的核心—XNU内核决定当前使用的线程数，只生成所需的线程执行处理。当处理结束，处理数减少，XNU内核结束不在需要的线程。

## dispatch_queue_create

- 通过 dispatch_queue_create 函数可生成 Dispatch Queue。生成的队列用 dispatch_queue_t 类型的变量接收。

```objective-c
dispatch_queue_t mySerialDispatchQueue = dispatch_queue_create("生成队列的名称", NULL);

第一个参数：
指定生成返回的Dispatch Queue的名称（可为NULL）
命名规则：简单易懂
该名称在Xcode和Instruments的调试器中作为Dispatch Queue的名称表示
该名称也会出现在程序崩溃时所生成的CrashLog中

第二个参数：
指定为NULL或DISPATCH_QUEUE_SERIAL,生成Serial Dispatch Queue。
指定为DISPATCH_QUEUE_CONCURRENT，生成Concurrent Dispatch Queue。

返回值：
表示Dispatch Queue的dispatch_queue_t类型
```

- 生成多个Serial Dispatch Queue时，各个Serial Dispatch Queue将并行执行，一个Serial Dispatch Queue只能生成一个线程，过多的使用Serial Dispatch Queue 会消耗大量内存，降低系统响应性能。

- 引用场景：

  - 多个线程更新相同资源导致的数据竞争问题是使用 Serial Dispatch Queue 

    ![](http://ww1.sinaimg.cn/large/006tNc79ly1g5tihvm5x7j30yb0hutgw.jpg)

  - 当想并行执行不发生数据竞争等问题的处理时，使用Concurrent Dispatch Queue，Concurrent Dispatch Queue由XNU管理线程，不会发生Serial Dispatch Queue引发的问题。

- iOS6之前，Dispatch Queue 需要进行手动管理释放，现已纳入ARC管理范围。

## Main Dispatch Queue / Global Dispatch Queue

- 除了上面通过API创建Dispatch Queue的方法，也可以获取系统标准提供的Dispatch Queue。

- **Main Dispatch Queue**：

  是在主线程中执行的Dispatch Queue。因为主线程只有一个，所以它是Serial Dispatch Queue。追加到Main Dispatch Queue的处理在主线程的RunLoop中执行。

  ```objective-c
  // 主队列的获取方法
  dispatch_queue_t queue = dispatch_get_main_queue();
  ```

- **Global Dispatch Queue**：
  是所有应用程序都能够使用的Concurrent Dispatch Queue。
  有四个执行优先级，高优先级（High Priority）、默认优先级（Default Priority）、低优先级（Low Priority）和后台优先级（Background Priority）。
  XNU内核管理的用于Global Dispatch Queue的线程，将各自使用的Global Dispatch Queue的执行优先级作为线程的执行优先级使用。
  向Global Dispatch Queue中追加处理时，要选择与处理内容对应的执行优先级的Global Dispatch Queue。
  
  ```objective-c
  // 全局并发队列的获取方法
  //第一个参数表示队列优先级，一般用 DISPATCH_QUEUE_PRIORITY_DEFAULT。
  //第二个参数暂时没用（未来做扩展用），用 0 即可。
  dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
  ```
  
- Dispatch Queue的种类：![](http://ww1.sinaimg.cn/large/006tNc79ly1g5tjm0dfswj3114093guo.jpg)

- [Concurrent Programming: APIs and Challenges](https://www.objc.io/issues/2-concurrency/concurrency-apis-and-pitfalls/)中的一张图片可以很直观地描述GCD与线程之间的关系：![img](https://upload-images.jianshu.io/upload_images/1966287-d52e018c156f0145.png)

## dispatch_set_target_queue
- dispatch_queue_create创建的Dispatch Queue都是使用默认优先级别的线程。

- 问题一、dispatch_set_target_queue的作用？
  1、变更dispatch_queue_create函数生成的Dispatch Queue的执行优先级
  
  ```objective-c
  dispatch_set_target_queue(dispatch_object_t object, dispatch_queue_t queue);
  
  第一个参数：
  要变更执行优先级的Dispatch Queue，不能为Main Dispatch Queue和Global Dispatch Queue。
  
  第二个参数：
  要使用的执行优先级相同优先级的Global Dispatch Queue。
  ```
  
  变更优先级
  
  ```objective-c
  // 要变更优先级的串行队列，初始是默认优先级
  dispatch_queue_t serialQueue = dispatch_queue_create("com.gcd.setTargetQueue.serialQueue", NULL);
  // 获取优先级为后台优先级的全局队列
  dispatch_queue_t globalDispatchQueueBackground = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
  // 变更优先级
  dispatch_set_target_queue(serialQueue, globalDispatchQueueBackground);
  ```
  
  2、作为Dispatch Queue的执行阶层。
  多个`Serial Dispatch Queue`用`dispatch_set_target_queue`函数指定目标（第二个参数）为某一个`Serial Dispatch Queue`。那**原先并行执行**的多个`Serial Dispatch Queue`，在目标`Serial Dispatch Queue`上只能同时执行一个处理，这样可以防止处理并行执行。
  
  
  ```objective-c
  设置执行阶层前：
  
  dispatch_queue_t serialQueue1 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue1", NULL);
  dispatch_queue_t serialQueue2 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue2", NULL);
  dispatch_queue_t serialQueue3 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue3", NULL);
  dispatch_queue_t serialQueue4 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue4", NULL);
  dispatch_queue_t serialQueue5 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue5", NULL);
  
  dispatch_async(serialQueue1, ^{
      NSLog(@"1");
  });
  dispatch_async(serialQueue2, ^{
      NSLog(@"2");
  });
  dispatch_async(serialQueue3, ^{
      NSLog(@"3");
  });
  dispatch_async(serialQueue4, ^{
      NSLog(@"4");
  });
  dispatch_async(serialQueue5, ^{
      NSLog(@"5");
  });
  
  输出：
  2018-07-22 18:37:51.908107+0800 Demo[1801:121342] 2
  2018-07-22 18:37:51.908107+0800 Demo[1801:121343] 3
  2018-07-22 18:37:51.908107+0800 Demo[1801:121341] 1
  2018-07-22 18:37:51.908148+0800 Demo[1801:121344] 4
  2018-07-22 18:37:51.908169+0800 Demo[1801:121352] 5
  ```
  
  ```objective-c
设置执行阶层后：
	  
	dispatch_queue_t serialQueue1 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue1", NULL);
	  dispatch_queue_t serialQueue2 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue2", NULL);
	  dispatch_queue_t serialQueue3 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue3", NULL);
	  dispatch_queue_t serialQueue4 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue4", NULL);
	  dispatch_queue_t serialQueue5 = dispatch_queue_create("com.gcd.setTargetQueue2.serialQueue5", NULL);
	
	//创建目标串行队列
	  dispatch_queue_t targetSerialQueue = dispatch_queue_create("com.gcd.setTargetQueue2.targetSerialQueue", NULL);
	
	//设置执行阶层
	  dispatch_set_target_queue(serialQueue1, targetSerialQueue);
	  dispatch_set_target_queue(serialQueue2, targetSerialQueue);
	  dispatch_set_target_queue(serialQueue3, targetSerialQueue);
	  dispatch_set_target_queue(serialQueue4, targetSerialQueue);
	  dispatch_set_target_queue(serialQueue5, targetSerialQueue);
	
	dispatch_async(serialQueue1, ^{
	      NSLog(@"1");
	  });
	  dispatch_async(serialQueue2, ^{
	      NSLog(@"2");
	  });
	  dispatch_async(serialQueue3, ^{
	      NSLog(@"3");
	  });
	  dispatch_async(serialQueue4, ^{
	      NSLog(@"4");
	  });
	  dispatch_async(serialQueue5, ^{
	      NSLog(@"5");
	  });
	
	  输出：
	  2018-07-22 18:41:30.403676+0800 Demo[1843:123760] 1
	  2018-07-22 18:41:30.403893+0800 Demo[1843:123760] 2
	  2018-07-22 18:41:30.405137+0800 Demo[1843:123760] 3
	  2018-07-22 18:41:30.407517+0800 Demo[1843:123760] 4
	  2018-07-22 18:41:30.409133+0800 Demo[1843:123760] 5
	```

## dispatch_after

在指定时间后执行处理，可以使用dispatch_after。

```objective-c
//dispatch_time函数获取从第一个参数指定时间开始，到第二个指定时间后的时间，通常用于计算相对时间。
//DISPATCH_TIME_NOW 表示现在的时间。
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 3ull * NSEC_PER_SEC);

// 在3秒后追加Block到Main Dispatch Queue中
dispatch_after(time, dispatch_get_main_queue(), ^{
    
    NSLog(@"waited at least three seconds");;
});

dispatch_after函数
第一个参数:
指定时间的dispatch_time_t类型的值，可以使用dispatch_time函数或dispatch_walltime函数作成。
  //dispatch_walltime函数:由POSIX中使用的struct timespec类型的时间得到dispatch_time_t类型的值，一般用于计算绝对时间，如指定某年某月某日某分某秒这一绝对时间。

第二个参数：
要追加处理的Dispatch Queue。

第三个参数：
要执行处理的Block。
```

- dispatch_after函数理解的注意事项？
   dispatch_after函数不是在指定时间后执行处理，而只是在指定时间追加处理**到Dispatch Queue**（第二个参数）。

- dispatch_after函数精度问题？
   有误差，大致延迟执行处理，可以用该函数，严格要求时间下会出现问题。

## Dispatch Group

如果遇到这样到需求：全部处理完多个预处理任务(block_1 ~ 4)后执行某个任务（block_finish），我们有两个方法：

- 如果预处理任务需要一个接一个的执行：将所有需要先处理完的任务追加到Serial Dispatch Queue中，并在最后追加最后处理的任务(block_finish)。
- 如果预处理任务需要并发执行：需要使用dispatch_group函数，将这些预处理的block追加到global dispatch queue中。（适用场景还有如果使用Concurrent Dispatch Queue 或同时使用多个dispatch queue时）

分别详细讲解一下两种需求的实现方式：

### 预处理任务需要一个接一个的执行：

这个需求的实现方式相对简单一点，只要将所有的任务（block_1 ~ 4 + block_finish）放在一个串行队列中即可，因为都是按照顺序执行的，只要不做多余的事情，这些任务就会乖乖地按顺序执行。

### 预处理任务需要并发执行：

```objective-c
dispatch_group_t group = dispatch_group_create();
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
for (NSInteger index = 0; index < 5; index ++) {
        dispatch_group_async(group, queue, ^{
            NSLog(@"任务%ld",index);
        });
}
    
dispatch_group_notify(group, queue, ^{
    NSLog(@"最后的任务");
});
复制代码
```

输出：

```objective-c
gcd_demo[40905:3057237] 任务0
gcd_demo[40905:3057235] 任务1
gcd_demo[40905:3057234] 任务2
gcd_demo[40905:3057253] 任务3
gcd_demo[40905:3057237] 任务4
gcd_demo[40905:3057237] 最后的任务
复制代码
```

因为这些预处理任务都是追加到global dispatch queue中的，所以这些任务的执行任务的顺序是不定的。但是最后的任务一定是最后输出的。

> dispatch_group_notify函数监听传入的group中任务的完成，等这些任务全部执行以后，再将第三个参数（block）追加到第二个参数的queue（相同的queue）中。

## dispatch_ group_wait

dispatch_group_wait 也是配合dispatch_group 使用的，利用这个函数，我们可以设定group内部所有任务执行完成的超时时间。

- 一旦调用dispatch_group_wait函数，该函数就处于调用状态而不返回。执行该函数的当前线程停止，在经过该函数中指定的时间或属于`Dispatch Group`的所有处理全部执行结束之前，执行该函数的线程停止。

```objective-c
- (void)dispatch_wait_1
{
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    for (NSInteger index = 0; index < 5; index ++) {
        dispatch_group_async(group, queue, ^{
            for (NSInteger i = 0; i< 1000000000; i ++) {
                
            }
            NSLog(@"任务%ld",index);
        });
    }
    
    dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull * NSEC_PER_SEC);
    long result = dispatch_group_wait(group, time);
    if (result == 0) {
        
        NSLog(@"group内部的任务全部结束");
        
    }else{
        
        NSLog(@"虽然过了超时时间，group还有任务没有完成");
    }
    
}

说明:
dispatch_group_wait函数返回值为0：全部处理执行结束
dispatch_group_wait函数返回值不为0：某一个处理还在执行中
等待时间为DISPATCH_TIME_FOREVER，由dispatch_ group_wait函数返回时，返回值恒为0。
```

输出：

```objective-c
gcd_demo[41277:3087481] 虽然过了超时时间，group还有任务没有完成，结果是判定为超时
gcd_demo[41277:3087563] 任务0
gcd_demo[41277:3087564] 任务2
gcd_demo[41277:3087579] 任务3
gcd_demo[41277:3087566] 任务1
gcd_demo[41277:3087563] 任务4
```

## dispatch_barrier_async

顾名思义，就是给处理之间加了一个“栅栏”，分割开栅栏两边的操作。

![](http://ww3.sinaimg.cn/large/006y8mN6ly1g6vxhp92i7j30lq0c7tau.jpg)

关于解决数据竞争的方法：读取处理是可以并发的，但是写入处理却是不允许并发执行的。

所以合理的方案是这样的：

- 读取处理追加到concurrent dispatch queue中
- 写入处理在任何一个读取处理没有执行的状态下，追加到serial dispatch queue中（也就是说，在写入处理结束之前，读取处理不可执行）。

我们看看如何使用dispatch_barrier_async来解决这个问题。

示例代码：

```objective-c
- (void)dispatch_barrier
{
    dispatch_queue_t meetingQueue = dispatch_queue_create("com.meeting.queue", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(meetingQueue, ^{
        NSLog(@"总裁查看合同");
    });
    
    dispatch_async(meetingQueue, ^{
        NSLog(@"董事1查看合同");
    });
    
    dispatch_async(meetingQueue, ^{
        NSLog(@"董事2查看合同");
    });
    
    dispatch_async(meetingQueue, ^{
        NSLog(@"董事3查看合同");
    });
    
    dispatch_barrier_async(meetingQueue, ^{
        NSLog(@"总裁签字");
    });
    
    dispatch_async(meetingQueue, ^{
        NSLog(@"总裁审核合同");
    });
    
    dispatch_async(meetingQueue, ^{
        NSLog(@"董事1审核合同");
    });
    
    dispatch_async(meetingQueue, ^{
        NSLog(@"董事2审核合同");
    });
    
    dispatch_async(meetingQueue, ^{
        NSLog(@"董事3审核合同");
    });
}
```

输出结果：

```objective-c
gcd_demo[41791:3140315] 总裁查看合同
gcd_demo[41791:3140296] 董事1查看合同
gcd_demo[41791:3140297] 董事3查看合同
gcd_demo[41791:3140299] 董事2查看合同
gcd_demo[41791:3140299] 总裁签字
gcd_demo[41791:3140299] 总裁审核合同
gcd_demo[41791:3140297] 董事1审核合同
gcd_demo[41791:3140296] 董事2审核合同
gcd_demo[41791:3140320] 董事3审核合同
```

> 在这里，我们可以将meetingQueue看成是会议的时间线。总裁签字这个行为相当于写操作，其他都相当于读操作。使用dispatch_barrier_async以后，之前的所有并发任务都会被dispatch_barrier_async里的任务拦截掉，就像函数名称里的“栅栏”一样。

## dispatch_apply

通过dispatch_apply函数，我们可以按照指定的次数将block追加到指定的队列中。并等待全部处理执行结束。

```objective-c
- (void)dispatch_apply_1
{
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_apply(10, queue, ^(size_t index) {
        NSLog(@"%ld",index);
    });
    NSLog(@"完毕");
}
复制代码
gcd_demo[6128:240332] 1
gcd_demo[6128:240331] 0
gcd_demo[6128:240334] 2
gcd_demo[6128:240332] 4
gcd_demo[6128:240334] 6
gcd_demo[6128:240331] 5
gcd_demo[6128:240332] 7
gcd_demo[6128:240334] 8
gcd_demo[6128:240331] 9
gcd_demo[6128:240259] 3
gcd_demo[6128:240259] 完毕
```

我们也可以用这个函数来遍历数组，取得下标进行操作:

```objective-c
- (void)dispatch_apply_2
{
    NSArray *array = @[@1,@10,@43,@13,@33];
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_apply([array count], queue, ^(size_t index) {
        NSLog(@"%@",array[index]);
    });
    NSLog(@"完毕");
}
```

输出：

```objective-c
gcd_demo[6180:244316] 10
gcd_demo[6180:244313] 1
gcd_demo[6180:244316] 33
gcd_demo[6180:244314] 43
gcd_demo[6180:244261] 13
gcd_demo[6180:244261] 完毕
```

我们可以看到dispatch_apply函数与dispatch_sync函数同样具有阻塞的作用（dispatch_apply函数返回后才打印完毕）。

我们也可以在dispatch_async函数里执行dispatch_apply函数：

```objective-c
- (void)dispatch_apply_3
{
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_async(queue, ^{
        
        NSArray *array = @[@1,@10,@43,@13,@33];
        __block  NSInteger sum = 0;
    
        dispatch_apply([array count], queue, ^(size_t index) {
            NSNumber *number = array[index];
            NSInteger num = [number integerValue];
            sum += num;
        });
        
        dispatch_async(dispatch_get_main_queue(), ^{
            //回到主线程，拿到总和
            NSLog(@"完毕");
            NSLog(@"%ld",sum);
        });
    });
}
```

## dispatch_suspend/dispatch_resume

挂起函数调用后对已经执行的处理没有影响，但是追加到队列中但是尚未执行的处理会在此之后停止执行。

```objective-c
dispatch_suspend(queue);
dispatch_resume(queue);
```

## Dispatch Semaphore

当并行执行的处理更新数据时，会产生数据不一致，甚至出现异常结束的情况，使用Serial Dispatch Queue和dispatch_barrier_async函数可避免，但使用Dispatch Semaphore可以进行更细颗粒的排他控制。

```cpp
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

NSMutableArray *array = [[NSMutableArray alloc] init];

for (int i = 0; i < 1000; i++) {
    dispatch_async(queue, ^{
        [array addObject:[NSNumber numberWithInt:i]];
    });
}

说明：可能出现同时访问数组，造成异常结束的问题。
```

- Dispatch Semaphore是持有计数的信号，该计数是多线程编程中的计数类型信号。
- Dispatch Semaphore中，计数为0时等待，计数为1或者大于1时，减去1而不等待。

- dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
  说明：
  - 该函数等待第一个参数semaphore的计数值达到大于等于1。
  - 当计数值大于等于1，或者在待机（等待的时间）中计数值大于等于1，对该计数进行减1并从该函数返回。
  - 第二个参数是dispatch_time_t类型值，与dispatch_group_wait函数相同。
    DISPATCH_TIME_FOREVER：永久等待。

- dispatch_semaphore_signal(semaphore);

  说明：
  处理结束时通过该函数将semaphore的计数值加1

- dispatch_semaphore_wait函数的返回值与dispatch_group_wait函数相同，semaphore大于等于1，result为0，semaphore为0，result不为0。可以通过返回值进行分支处理。

  ```cpp
  dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);

  dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull * NSEC_PER_SEC);

  long result = dispatch_semaphore_wait(semaphore, time);

  if (result == 0) {
      // 由于semaphore的计数值大于等于1或在等待时间内计数值大于等于1， 所以semaphore的计数值减1 执行需要进行排他控制的处理
  } else {
      // semaphore的计数值直到1秒的等待时间结束都为0。
  }
  ```

对数组的安全处理：

```cpp
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);

NSMutableArray *array = @[].mutableCopy;

for (int i = 0; i < 1000; i++) {
    dispatch_async(queue, ^{
        
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        
        [array addObject:[NSNumber numberWithInt:i]];
        
        // 处理结束时通过该函数将semaphore的计数值加1
        dispatch_semaphore_signal(semaphore);
    });
}
```

## dispatch_once

通过dispatch_once处理的代码只执行一次，而且是线程安全的（常用于单例模式初始化）：

```objectivec
static int initialized = NO;
if (initialized == NO) {
    // 初始化
    initialized = YES;
}

说明：
上面代码，在大多数情况下是安全的，但是在多核CPU中，在正在更新表示是否初始化的标志变量initialized时读取，就有可能多次执行初始化处理
static dispatch_once_t pred;
dispatch_once(&pred, ^{
    //初始化
});
```



**参考博客**：[iOS 多线程：『GCD』详尽总结](https://bujige.net/blog/iOS-Complete-learning-GCD.html)