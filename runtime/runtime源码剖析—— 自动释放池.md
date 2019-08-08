[TOC]

- autoreleasepool 的功能、特性，用法都在之前内存管理的文章中说过了，这篇博客主要站在源码的角度来分析一下 autoreleasepool 到底是什么。

# autoreleasepool究竟是什么
新建一个iOS项目，用clang重写一下main.m（在命令行中使用 `clang -rewrite-objc main.m` 让编译器重新改写这个文件）。在main.cpp文件的最后，看到main函数变成了这样：

```objective-c
// 原本iOS项目的main.m
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}

//clang重新改写以后的main.m
int main(int argc, char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        return UIApplicationMain(argc, argv, __null, NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class"))));
    }
}
```
也就是说 `@autoreleasepool {}` 被转换为一个 `__AtAutoreleasePool` 结构体：

```objective-c
{
    __AtAutoreleasePool __autoreleasepool;
}
```

为了弄清这行代码的意义，我们搜索一下__AtAutoreleasePool，它的定义是这样的：
```objective-c
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```

这个结构体会在初始化时调用 `objc_autoreleasePoolPush()` 方法，会在析构时调用 `objc_autoreleasePoolPop` 方法。

这表明，我们的 `main` 函数在实际工作时其实是这样的：

```objective-c
int main(int argc, const char * argv[]) {
    {
        void * atautoreleasepoolobj = objc_autoreleasePoolPush();
        
        // do whatever you want
        
        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }
    return 0;
}
```

`@autoreleasepool` 只是帮助我们少写了这两行代码而已，让代码看起来更美观，然后要根据上述两个方法来分析自动释放池的实现。

# AutoreleasePoolPage 

这里我们看一下上面提到的 `objc_autoreleasePoolPush` 和 `objc_autoreleasePoolPop`方法的实现：

```objective-c
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}

void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);
}
```
- 方法中出现了`AutoreleasePoolPage`的概念，`AutoreleasePoolPage` 是一个 C++ 中的类：
![](https://imgconvert.csdnimg.cn/aHR0cDovL3d3My5zaW5haW1nLmNuL2xhcmdlLzAwNnROYzc5bHkxZzU0Mm0xeHA3M2ozMDh6MGNpbXh3LmpwZw)

它在 NSObject.mm 中的定义是这样的：
```objective-c
class AutoreleasePoolPage {
    magic_t const magic; //用于对当前 AutoreleasePoolPage 完整性的校验
    id *next;
    pthread_t const thread;//保存了当前页所在的线程
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
};
```
- **每一个自动释放池都是由一系列的 AutoreleasePoolPage 组成的，并且每一个 AutoreleasePoolPage 的大小都是 4096 字节（16 进制 0x1000）**
```objective-c
#define I386_PGBYTES 4096
#define PAGE_SIZE I386_PGBYTES
```
## 双向链表
自动释放池中的 AutoreleasePoolPage 是以双向链表的形式连接起来的：

![](https://imgconvert.csdnimg.cn/aHR0cDovL3d3Mi5zaW5haW1nLmNuL2xhcmdlLzAwNnROYzc5bHkxZzU0Mm42bXU5M2ozMHQyMDVyZGdkLmpwZw)

>parent 和 child 就是用来构造双向链表的指针。

## 自动释放池中的栈
如果我们的一个 AutoreleasePoolPage 被初始化在内存的 0x100816000 ~ 0x100817000 中，它在内存中的结构如下：

![](http://ww3.sinaimg.cn/large/006tNc79ly1g5s0micf0sj30f20jyjsz.jpg)

其中有 56 bit 用于存储 AutoreleasePoolPage 的成员变量，剩下的 0x100816038 ~ 0x100817000 都是用来存储加入到自动释放池中的对象。

> begin() 和 end() 这两个类的实例方法帮助我们快速获取 0x100816038 ~ 0x100817000 这一范围的边界地址。

next 指向了下一个为空的内存地址，如果 next 指向的地址加入一个 object，它就会如下图所示移动到下一个为空的内存地址中：

> 关于 hiwat 和 depth 在文章中并不会进行介绍，因为它们并不影响整个自动释放池的实现，也不在关键方法的调用栈中。
>

## POOL_BOUNDARY（哨兵对象）
- **POOL_BOUNDARY(在上图中是POOL_SENTINEL)就是哨兵对象，它是一个宏，值为nil，标志着一个自动释放池的边界**
- 在每个自动释放池初始化调用 `objc_autoreleasePoolPush` 的时候，都会把一个 `POOL_BOUNDARY` push 到自动释放池的栈顶，并且返回这个 `POOL_BOUNDARY` 哨兵对象。
- 而当方法 objc_autoreleasePoolPop 调用时，就会向自动释放池中的对象发送 release 消息，直到第一个 POOL_SENTINEL：
![](https://imgconvert.csdnimg.cn/aHR0cDovL3d3Mi5zaW5haW1nLmNuL2xhcmdlLzAwNnROYzc5bHkxZzU0M2RlbDUwMmozMHlnMGVldDlsLmpwZw)

## objc_autoreleasePoolPush 方法

了解了 `POOL_BOUNDARY`，我们来重新回顾一下 `objc_autoreleasePoolPush` 方法：

```objective-c
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}
```

它调用 `AutoreleasePoolPage` 的类方法 `push`，也非常简单：

```objective-c
static inline void *push() {
   return autoreleaseFast(POOL_BOUNDARY);
}
```

在这里会进入一个比较**关键**的方法 `autoreleaseFast`，并传入哨兵对象 `POOL_BOUNDARY`：

```objective-c
static inline id *autoreleaseFast(id obj)
{
   AutoreleasePoolPage *page = hotPage();
   if (page && !page->full()) {
       return page->add(obj);
   } else if (page) {
       return autoreleaseFullPage(obj, page);
   } else {
       return autoreleaseNoPage(obj);
   }
}
```

上述方法分三种情况选择不同的代码执行：

- 有`hotPage`并且当前`page`不满
	- 调用 `page->add(obj)` 方法将对象添加至 `AutoreleasePoolPage` 的栈中
- 有`hotPage`并且当前`page`已满
  - 调用 `autoreleaseFullPage` 初始化一个新的页
  - 调用 `page->add(obj)` 方法将对象添加至 `AutoreleasePoolPage` 的栈中
- 无`hotPage`
  - 调用 `autoreleaseNoPage` 创建一个 `hotPage`(只有这个方法会在page中插入一个POOL_BOUNDARY)
  - 调用 `page->add(obj)` 方法将对象添加至 `AutoreleasePoolPage` 的栈中

最后的都会调用 `page->add(obj)` 将对象添加到自动释放池中。

> `hotPage` 可以理解为当前正在使用的 `AutoreleasePoolPage`。

### page->add 添加对象
id *add(id obj) 将对象添加到自动释放池页中：

```objective-c
id *add(id obj) {
    id *ret = next;
    *next = obj;
    next++;
    return ret;
}
```

>笔者对这个方法进行了处理，更方便理解。

这个方法其实就是一个压栈的操作，将对象加入 AutoreleasePoolPage 然后移动栈顶的指针。

### autoreleaseFullPage（当前 hotPage 已满）
`autoreleaseFullPage` 会在当前的 `hotPage` 已满的时候调用：
```objective-c
static id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page) {
    do {
        if (page->child) page = page->child;
        else page = new AutoreleasePoolPage(page);
    } while (page->full());

    setHotPage(page);
    return page->add(obj);
}
```
它会从传入的 page 开始遍历整个双向链表，直到：

1. 查找到一个未满的`AutoreleasePoolPage`
2. 使用构造器传入`parent`创建一个新的AutoreleasePoolPage
在查找到一个可以使用的`AutoreleasePoolPage`之后，会将该页面标记成`hotPage`，然后调动上面分析过的`page->add`方法添加对象。

### autoreleaseNoPage（没有 hotPage)
如果当前内存中不存在 `hotPage`，就会调用 `autoreleaseNoPage` 方法初始化一个`AutoreleasePoolPage`：
```objective-c
static id *autoreleaseNoPage(id obj) {
    AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
    setHotPage(page);
    if (obj != POOL_SENTINEL) {
        page->add(POOL_BOUNDARY);
    }
    return page->add(obj);
}
```
既然当前内存中不存在 `AutoreleasePoolPage`，就要从头开始构建这个自动释放池的双向链表，也就是说，新的 `AutoreleasePoolPage` 是没有 `parent` 指针的。

初始化之后，将当前页标记为 `hotPage`，然后会先向这个 `page` 中添加一个 `POOL_BOUNDARY` 对象，来确保在 `pop` 调用的时候，不会出现异常。

最后，将 obj 添加到自动释放池中。

## objc_autoreleasePoolPop 方法

同样，回顾一下上面提到的 `objc_autoreleasePoolPop` 方法：

```objective-c
void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);
}
```

> 看起来传入任何一个指针都是可以的，但是在整个工程并没有发现传入其他对象的例子。不过在这个方法中**传入其它的指针也是可行的**，会将自动释放池释放到相应的位置。

我们一般都会在这个方法中传入一个哨兵对象 `POOL_BOUNDARY`

``AutoreleasePoolPage::pop` 方法的调用：

```objective-c
static inline void pop(void *token) {
    AutoreleasePoolPage *page = pageForPointer(token);
    id *stop = (id *)token;

    page->releaseUntil(stop);

    if (page->child) {
        if (page->lessThanHalfFull()) {
            page->child->kill();
        } else if (page->child->child) {
            page->child->child->kill();
        }
    }
}
```

> 在这个方法中删除了大量无关的代码，以及对格式进行了调整。

该静态方法总共做了三件事情：

1. 使用 `pageForPointer` 获取当前 `token` 所在的 `AutoreleasePoolPage`
2. 调用 `releaseUntil` 方法释放**栈中的**对象，直到 `stop`
3. 调用 `child` 的 `kill` 方法

> 如果当前page小于一半满，则把当前页的所有孩子都杀掉，否则，留下一个孩子，从孙子开始杀。
>
> 为什么呢？Apple假设，当前page一半都没满，说明已经够了，把接下来的全kill，如果超过一半，就认为下一页还有存在的必要，所以从孙子开始杀
>
> 因为孩子页的东西已经都被释放掉了，留下来可以节约创建操作的花费

### token是什么
先来看下Apple在Autorelease pool implementation中写的注释
```objective-c
/***********************************************************************
 自动释放池实现
 
 一个线程的自动释放池是一个指针堆栈。
每个指针要么指向要被释放的对象，要么是POOL_BOUNDARY说明一个pool的边界

token是指向该pool的POOL_BOUNDARY的指针。什么时候池被pop，所有在哨兵对象前的hotter对象都被释放。

pool被分成一个双向指针构成的pages。pages在必要的时候被添加和删除
线程本地存储指针指向hot page，在这里新被autoreleased的objects被存储


   Autorelease pool implementation

   A thread's autorelease pool is a stack of pointers. 
   Each pointer is either an object to release, or POOL_BOUNDARY which is 
     an autorelease pool boundary.
   A pool token is a pointer to the POOL_BOUNDARY for that pool. When 
     the pool is popped, every object hotter than the sentinel is released.
   The stack is divided into a doubly-linked list of pages. Pages are added 
     and deleted as necessary. 
   Thread-local storage points to the hot page, where newly autoreleased 
     objects are stored. 
**********************************************************************/
```
- 这里就讲清楚了toekn本质就是指向POOL_BOUNDARY的指针，存储着每次push时插入的POOL_BOUNDARY的地址

- token是在调用 `push`方法时内部return的`dest`赋的值，`push`方法只有创建自动释放池时会调用

- 看一下上文中`__AtAutoreleasePool`的定义就了解到了它是如何将token指针传给pop方法的：

  ```objective-c
  struct __AtAutoreleasePool {
    __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
    ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
    void * atautoreleasepoolobj;
  };
  ```

### releaseUntil
- 这个方法顾名思义，就是将对象一直release，一直到stop【token】
releaseUntil 方法的实现如下：
```objective-c
void releaseUntil(id *stop) {
    while (this->next != stop) {
        AutoreleasePoolPage *page = hotPage();
        while (page->empty()) {
            page = page->parent;
            setHotPage(page);
        }
        page->unprotect();
        id obj = *--page->next;
        memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
        page->protect();

        if (obj != POOL_BOUNDARY) {
            objc_release(obj);
        }
    }
    setHotPage(this);
}
```
用一个 while 循环持续释放 AutoreleasePoolPage 中的内容，直到 next 指向了 stop 。
使用 memset 将内存的内容设置成 SCRIBBLE，然后使用 objc_release 释放对象。

# 总结

1. autoreleasepool部分源码中，POOL_BOUNDARY 就是哨兵对象，它是一个宏，值为nil，标志着一个自动释放池的边界。

2. 一页autoreleasepoolpage的存储信息，一个AutoreleasePoolPage占据4096个字节，扣除56个字节存储下面的信息外，其余的都用来存储加入到page中的对象地址。


3. autoreleasepoolpage 是以双向链表的形式连接起来的，只有新建一个pool会执行push操作，这时候会在page中插入一个POOL_BOUNDARY，并不是每页都插。
4. 执行pop操作时，先调用 `releaseUntil` 方法释放**栈中的**对象，直到 `stop`；再调用 `child` 的 `kill` 方法，如果当前page小于一半满，则把当前页的所有孩子都杀掉，否则，留下一个孩子，从孙子开始杀。

最后借用实验室伙计的“神图”总结一下这篇博客：

![](http://ww1.sinaimg.cn/large/006tNc79ly1g5sa2g6qqlj30y90h4q73.jpg)