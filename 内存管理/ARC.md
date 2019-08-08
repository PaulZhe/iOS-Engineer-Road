[TOC]

# 前言

- 什么是ARC？

  - ARC 是 Automatic Reference Counting 的缩写, 即自动引用计数. 这是苹果在 iOS5 中引入的内存管理机制.不仅能够降低程序崩溃和内存泄露的风险, 而且可以减少开发者的工作量, 能够大幅度提升程序的 流畅性 和 可预测性 。

- ARC解决了MRC下内存管理的哪些问题？

  1. 当我们要释放一个堆内存时，首先要确定指向这个堆空间的指针都被release了。（避免提前释放）

  2. 释放指针指向的堆空间，首先要确定哪些指针指向同一个堆，这些指针只能释放一次。（MRC下即谁创建，谁释放，避免重复释放）

  3. 模块化操作时，对象可能被多个模块创建和使用，不能确定最后由谁去释放。

  4. 多线程操作时，不确定哪个线程最后使用完毕
  
- 上面这两个问题都是用很书面化的语言描述了ARC，唐巧大神的一段描述我觉得很有意思，贴出来方便我们直观地感受MRC时代到ARC时代的转变：
  
  > 我们先写好一段 iOS 的代码，然后屏住呼吸，开始运行它，不出所料，它崩溃了。在 MRC 时代，即使是最牛逼的 iOS 开发者，也不能保证一次性就写出完美的内存管理代码。于是，我们开始一步一步调试，试着打印出每个怀疑对象的引用计数（Retain Count），然后，我们小心翼翼地插入合理的 `retain` 和 `release` 代码。经过一次又一次的应用崩溃和调试，终于有一次，应用能够正常运行了！于是我们长舒一口气，露出久违的微笑。
  >
  > 是的，这就是那个年代的 iOS 开发者，通常情况下，我们在开发完一个功能后，需要再花好几个小时，才能把引用计数管理好。

# 内存管理的思考方式

- 首先还是老生常谈的内存管理四大原则
  	- 自己生成的对象，自己所持有
  	- 非自己生成的对象，自己也能持有
  	- 自己持有的对象不需要时释放
  	- 非自己生成的对象不再需要时释放
- 这种思考方式在ARC时代也是可行的，只不过再理解上增加一点点变化。
- 首先再重述一下我在[iOS内存管理——引用计数与MRC篇](https://blog.csdn.net/qq_42347755/article/details/97560226) 中写道的我理解的“持有”：**表示持有者还需使用这个对象，并要负责这个对象的释放。**这对理解这篇文章很重要。
- 至于理解上的变化是什么，我们先要理解ARC中追加的所有权声明。

# 所有权修饰符

- Objective-C 中将变量类型定义为 id 类型和各种对象类型，所谓对象类型就是指向 NSObject 这样的 Objective-C 类的指针，例如“NSObject *”。id 类型用于隐藏对象类型的类名部分，相当于C语言中常用的“void *”。
- ARC有效时，上述的 id 类型和各种对象类型必须附加所有权修饰符，所有权修饰符一共四种：
  - __strong 修饰符
  - __weak 修饰符
  - __unsafe_retained 修饰符
  - __autoreleasing 修饰符

## __strong 修饰符

- __strong 修饰符是 id 类型和各种对象类型默认的所有权修饰符。

- 附有__strong 修饰符的变量在obj超出其变量作用域时，即在该变量被废弃时，会释放其被赋值的对象。

- “strong”如其名所示，__strong修饰符表示对对象的“强引用”，持有强引用的变量在超出其作用域时被废弃，随着强引用失效，引用的对象随之被释放。

- 下面这个例子是内存管理思考方式第二条的应用：非自己生成的对象，自己也能持有

  ```objective-c
  {
    //取得非自己生成并持有的对象
    id __strong obj = [NSMutableArray];
    //因为变量obj是强引用，所以自己持有对象
    
  } //因为变量obj超出其作用域，强引用失效，所以自动地释放自己持有的对象
  ```

- 方法参数使用__strong修饰符的意义，先用参数对象obj强引用传进来的(NSObject对象)参数。

  ```objective-c
  - (void)setObject:(id __strong)obj {
  	_obj  = obj;
  }
  ```
  
- `__strong` 修饰符同后面要说的 `__weak修饰符`和` __autoreleasing修饰符一起`，可以保证将附有这些修饰符的自动变量**初始化**为nil

## __weak修饰符

- `__weak`修饰符的用途：避免循环引用。它与`__strong`修饰符相反，提供弱引用，弱引用不能持有对象示例，所以形不成下图这种相互持有或者环状持有的情况：

  ![img](https://img0.tuicool.com/67VBN3F.png!web)

  ![img](https://img0.tuicool.com/ZZvQb2f.png!web)

- `__weak修饰符`还有一个优点：在持有某对象的弱引用时，若该对象被废弃，则此弱引用将自动失效且处于被nil赋值的状态（空弱引用）。

- 神奇的打印试验：

  ```objective-c
  //在ARC中我们可以使用__bridge来查看应用计数
  
  NSObject *obj0 = [[NSObject alloc] init];
          printf("retain count = %ld\n",CFGetRetainCount((__bridge CFTypeRef)(obj0))); 
          NSObject * __weak obj1 = obj0;
          printf("retain count = %ld\n",CFGetRetainCount((__bridge CFTypeRef)(obj1)));
          printf("retain count = %ld\n",CFGetRetainCount((__bridge CFTypeRef)(obj0)));
          
  //        retain count = 1
  //        retain count = 2
  //        retain count = 1
  
  ```

  这里学习的时候困扰了一段时间，明明 obj1 和 obj0 指向同一块内存地址，打印出的引用计数却不一样，第一次得出结论是看到《Objective-C高级编程》中的一段观点：

  >  访问附有`__weak`修饰符的变量时就必定要访问注册到autoreleasepool中的对象，以此确保在访问该变量时该变量存在。
  >
  >  “使用附有__weak （NSLog(@"%@",obj1);）修饰符的变量，即是使用注册到autoreleasepool” 
  >
  >  { 
  >
  >  ​	id tmp = objc_loadWeakretained(&obj1);
  >
  >   ​    _objc_rootAutorelease(tmp); 
  >
  >  ​	NSLog(@"%@",tmp); 
  >
  >  }
  
  当时以为是注册到自动释放池时导致了引用计数加一，但这个结论现在看来漏洞百出，首先我在 [iOS内存管理——引用计数与MRC篇](https://blog.csdn.net/qq_42347755/article/details/97560226#autorelease_106) 中分析autorelease时说过了: 
  
  > 注意：autorelease 方法不会改变调用者的引用计数，它只是改变了对象释放时机，不再让程序员负责释放这个对象，而是交给自动释放池去处理。
  >
  
  注册到自动释放池并不会改变对象的引用计数，其次，《Objective-C高级编程》的观点也已经过时了，现在如下：
  
  ```objective-c
  { 
  	id temp = objc_loadWeakRetained(&weakObj);
  	NSLog(@"=====%@",temp);
  	objc_release(temp); 
  }
  ```
  
  “罪魁祸首”是这句代码 `objc_loadWeakRetained(&weakObj);`在runtime中找到其源码，如下：
  
  ```objective-c
  id
  objc_loadWeakRetained(id *location)
  {
      id obj;
      id result;
      Class cls;
  
      SideTable *table;
      
   retry:
      
      obj = *location; // 获取指向的对象
      if (!obj) return nil;
      if (obj->isTaggedPointer()) return obj;
      
      table = &SideTables()[obj];// 获取对象的SideTable
      
      table->lock(); // 加锁
      if (*location != obj) { // 对比一次，是在有其他地方在操作修改
          table->unlock();
          goto retry;
      }
      
      result = obj;
  
      cls = obj->ISA();
      if (! cls->hasCustomRR()) {
          // 没有自定义retain/release,调用系统的Retain
          assert(cls->isInitialized());
          if (! obj->rootTryRetain()) {
              // 如果retain失败，返回nil
              result = nil;
          }
      }
      else {
          // 有自定义的retain/release
          // 先看是否初始化initialize
          if (cls->isInitialized() || _thisThreadIsInitializingClass(cls)) {
              BOOL (*tryRetain)(id, SEL) = (BOOL(*)(id, SEL))
                  class_getMethodImplementation(cls, SEL_retainWeakReference);
              // 是否实现了retainWeakReference，
              if ((IMP)tryRetain == _objc_msgForward) {
                  result = nil;
              }
              else if (! (*tryRetain)(obj, SEL_retainWeakReference)) {
                  // 是否可以retain对象，返回NO，该变量将使用“nil”
                  result = nil;
              }
          }
          else {
              // 没有初始化，先初始化，goto retry
              table->unlock();
              _class_initialize(cls);
              goto retry;
          }
      }
          
      table->unlock();
      return result;
  }
  
  ```
  
  **最终结论**是：访问附有`__weak`修饰符的变量时会生成一个临时变量tmp，附有`__weak`修饰符的变量执行 `objc_loadWeakRetained(&weakObj);`使引用计数加一，再将其返回值赋给tmp，访问结束再将tmp释放掉，从结果上看就是访问该变量时引用计数多了1，但访问指向同一块内存的其他变量时引用计数是正常值。注意，整个流程不经过自动释放池。

## __unsafe_unretained修饰符

- `__unsafe_unretained修饰符`正如其名，是不安全的所有权修饰符。尽管ARC式的内存管理是编译器的工作，但附有`__unsafe_unretained修饰符`的变量不属于编译器的内存管理对象。

- 它的作用与`__weak修饰符`相似，提供弱引用，但它不能像`__weak修饰符`一样在对象被废弃时将修饰指针指向置nil，这样会造成悬垂指针的情况发生，所以说它是不安全的。在iOS4以后基本已被`__weak修饰符`替代。

- 这里涉及到一个立即释放对象的概念，一下源代码会引起编译器警告

  ```objective-c
  id __unsafe_unretained obj = [NSObject alloc] init];
  ```

  与`__weak修饰符`完全相同，这是由于编译器判断生成并持有的对象不能继续持有，从而发出警告。该源代码通过编译器转换为以下形式：

  ```objective-c
  /*编译器模拟代码*/
  id obj = obj_msgSend(NSObject, @selector(alloc));
  obj_msgSend(obj, @selector(init));
  obj_release(obj);
  ```

  obic_release函数立即释放了生成并持有的对象，如果是`__unsafe_retain`修饰的对象，这样该对象的悬垂指针被赋值给变量obj中。

  如果最初不赋值变量又会如何呢？下面源代码在MRC时必定会发生内存泄漏：

  ```objective-c
  [[NSObject alloc] init];
  ```

  虽然没有指定赋值变量，但与上述源代码完全相同。由于不能继续持有生成并持有的对象，所以编译器生成了立即调用`objc_release`函数的源代码。而由于ARC的处理，这样的源代码也不会造成内存泄露。

  另外，可以调用被立即释放的对象的实例方法。

## __autoreleasing修饰符

- 对象赋值给附有`__autoreleasing修饰符`的变量等价于在ARC无效时调用对象的 autorelease 方法，即对象注册到aureleasepool。可以理解为在ARC有效时，@autoreleasepool块替代NSAutoreleasePool类，用附有`__autoreleasing修饰符`的变量替代 autorelease 方法。

  ![](http://ww2.sinaimg.cn/large/006tNc79ly1g5jdchvjn5j311v0du4cd.jpg)

- 当时，显式地添加`__autoreleasing修饰符`同显式地附加`__strong修饰符`一样罕见。下面我们看看为什么非显式地使用`__autoreleasing修饰符`也可以。

	1. 取得非自己生成并持有的对象时，虽然可以使用alloc/new//copy/mutableCopy以外的方法来取得对象，但该对象已经被注册到autoreleasepool中了。这是由于编译器会检查方法名是否以alloc/new/copy/mutableCopy开始，如果不是则自动将返回值的对象注册到autorelasepool。维系这些规则所需的全部内存管理事宜均由ARC自动处理。

    - 但这时问题又来了，经过下面的这个打印试验，发现不管是不是以`alloc`等开头的方法，最终都没有执行`autorelease`方法， 以下两段代码差别只有自己手写的实例生成对象方法方法名不一样。
    ```objective-c
    @interface TestObj: NSObject 
    - (TestObj *)allocObj;
    @end
  @implementation TestObj
    - (TestObj *)allocObj {
      TestObj *obj = [[TestObj alloc] init];
        return obj;
    }
    @end
    int main(int argc, const char * argv[]) {
        @autoreleasepool {
            TestObj *test = [TestObj new];
            TestObj *test1 = [test allocObj];
        }
        return 0;
    }
    ```
    ```objective-c
    @interface TestObj: NSObject 
    - (TestObj *)Obj;
    @end
  @implementation TestObj
    - (TestObj *)Obj {
      TestObj *obj = [[TestObj alloc] init];
        return obj;
    }
    @end
    int main(int argc, const char * argv[]) {
        @autoreleasepool {
            TestObj *test = [TestObj new];
            TestObj *test1 = [test Obj];
        }
        return 0;
    }
    ```

    - 这里就涉及到了ARC简化引用计数的问题，ARC为了优化代码，会调用`objc_autoreleaseReturnValue`函数取代`autorelease`方法，此函数会检视当前方法返回之后即将要执行的那段代码。若发现那段代码要在返回的对象上执行retain操作，则设置全局数据结构（此数据结构具体内容因处理器而异）中的一个标志位，而不执行autorelease操作。`objc_autoreleaseReturnValue`函数的实现(伪代码)如下：

      ```objective-c
      id objc_autoreleaseReturnValue(id object) {
        if(/*caller will retain object*/) {
          self_flag(object);
          return object; ///< No autorelease
        } else {
          return [object autorelease];
        }
      }
      ```

    - 与此相似，如果方法返回了一个自动释放的对象，而调用方法的代码要保留此对象，那么此时不直接执行`retain`，而改为执行`objc_retainAutoreleasedReturnValue`函数。此函数要检测刚才提到的那个标志位，若已经置位，则不执行retain操作。`objc_autoreleaseReturnValue`函数的实现(伪代码)如下：
      ```objective-c
      id objc_autoreleaseReturnValue(id object) {
        if(get_flag（object）) {
          clear_flag(object);
          return object; ///< No retain
        } else {
          return [object retain];
        }
      }
      ```
    - 这样做的原因是设置并检测标志位，要比调用`autorelease`和`retain`更快，如果被赋值对象要持有方法返回值，设置标志位可以节省掉`autorelease`和`retain`这一对不必要的操作.
  
  2. id 的指针（`id __autoreleasing *obj`）或对象的指针（`NSObject *__autoreleasing *obj`）在没有显示指定时会被附加上`__autoreleasing`修饰符。
  
- 注意：赋值给对象指针时，所有权修饰符必须一致；另外，显式地指定`__autoreleasing修饰符`时，必须注意对象变量要为自动变量（包括局部变量、函数以及方法参数）。
  
- 利用`_objc_autoreleasePoolPrint()`这一函数可有效地帮助我们调试注册到 autoreleasepool上的对象。
  
# ARC规则

- 不能使用retain/release/retainCount/autorelease
-  不能使用 NSAllocateObject/NSDellocateObject
- 须遵守内存管理的方法命名规则
  - 这里单独将`init`方法拎出来说一说：该方法必须是实例方法，且必须要返回对象。返回的对象应为id类型或该方法声明类的对象类型，或是该类的超类型或子类型。该返回对象并不注册到`autoreleasepool`上。基本上只是对alloc方法返回值的对象进行初始化处理并返回该对象。
- 不要显式地调用dealloc
  - ARC有效时不必书写`[super dealloc]`。dealloc中只需记述废弃对象时所必须的处理。
- 使用@autoreleasepool块代替NSAutoreleasePool
- 不能使用区域（NSZone）
- 对象型变量不能作为C语言结构体（struct/union）的成员
- 显式转换“id”和“void *”