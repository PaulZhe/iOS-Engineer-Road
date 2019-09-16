[TOC]

***
# Block是什么
- Block是带有自动变量的匿名函数。如字面意思，Block没有函数名，另外Block带有插入记号"^"，插入记号便于查找到Block，后面跟的一个括号表示块所需要的一个参数列表。和函数一样，可以给块传递参数，并且也具有返回值。
- 不同点在于，块定义在函数或者方法内部，并且能够访**问在函数或方法范围**内的任何变量。通常情况下，这些变量能够访问但不能改变其值，有一个特殊的块修改器（由块前面由两个下划线字符组成）能够修改块内变量的值。
- Blocks 也被称作 **闭包**、**代码块**。展开来讲，Blocks 就是一个代码块，把你想要执行的代码封装在这个代码块里，等到需要的时候再去调用。

***
# Block的语法
## Block的语法格式

- Block的完整语法格式如下：
  `^ returnType (argument list) {expressions}`
  
- 与 C 语言函数的区别：
  1. 没有函数名
  2. 带有 “^” 符号
  
- 我们写一个完整的Block：

  ![](http://ww2.sinaimg.cn/large/006y8mN6ly1g6r3ttihn2j30g202ot95.jpg)
  
  ```objectivec
  ^ int (int x) { return x; }
  ```
  
- 也可以写省略格式的Block，比如省略返回值：

  ![](http://ww2.sinaimg.cn/large/006y8mN6ly1g6r3umonkej30g90840tp.jpg)

  ```objectivec
  ^ (int x) { return x; }
  ```
  Block省略返回值类型时，如果表达式中有return语句就使用该返回值的类型，没有return语句就使用void类型。

- 如果还没有参数也可以省略参数列表：

  ![](http://ww3.sinaimg.cn/large/006y8mN6ly1g6r3vlpw8gj30fo07ldgh.jpg)

  ```objectivec
  ^ { NSLog(@"Hello World"); }
  ```

## Block类型的声明与赋值的使用

### Block与一般的C语言变量相似的使用
Block类型变量与一般的C语言变量完全相同，可以作为自动变量，函数参数，静态变量，全局变量使用。比如将块定义在main函数外部，就可以将它扩展到全局范围。

- 先来看看C语言函数将是如何将所定义的函数的地址赋值给函数指针类型变量中。

  ```c
  int func (int count)
  {
      return count + 1;
  }
  
  int (*funcptr) (int) = &func;
  ```

- 代码中使用Block语法就相当于生成了可赋值给Block类型变量的值。
  
  ```objective-c
  // Block语法生成的值赋值给Block类型变量
  int (^blk) (int) = ^int (int count) {
      return count + 1;
  };
  int (^myBlock)(int) = blk; 
  ```
  
  与前面的使用函数指针的源代码对比可知，声明Block类型变量仅仅是将声明函数指针类型变量的"*"变为 "^"
  
- 在函数返回值中指定Block类型，可以将Block作为函数的返回值返回。（这个看起来比较特殊，单独拎出来写一下）
  
  ```objective-c
  int (^func()(int)) {
      return ^(int count){
          return count + 1;
    	}
  }
  ```
### Block在OC中的使用

- 作为带有 property 声明的成员变量：`@property (nonatomic, copy) 返回值类型 (^变量名) (参数列表);`

  ```objective-c
  /* Blocks 变量作为带有 property 声明的成员变量 */
  @property (nonatomic, copy) void (^myPropertyBlock) (void);
  
  // Blocks 变量作为带有 property 声明的成员变量
  - (void)useBlockAsProperty {
      self.myPropertyBlock = ^{
          NSLog(@"useBlockAsProperty");
      };
  
      self.myPropertyBlock();
  }
  ```

- 作为 OC 方法参数：`- (void)someMethodThatTaksesABlock:(返回值类型 (^)(参数列表)) 变量名;`

  ```objective-c
  // Blocks 变量作为 OC 方法参数
  - (void)someMethodThatTakesABlock:(void (^)(NSString *)) block {
      block(@"someMethodThatTakesABlock:");
  }
  ```

- 调用含有 Block 参数的 OC方法：`[someObject someMethodThatTakesABlock:^返回值类型 (参数列表) { 表达式}];`

  ```objective-c
  // 调用含有 Block 参数的 OC方法
  - (void)useBlockAsMethodParameter {
      [self someMethodThatTakesABlock:^(NSString *str) {
          NSLog(@"%@",str);
      }];
  }
  ```

###作为 typedef 声明类型

- 如果每次写 block 变量的时候都这样写，那不是很麻烦吗。我们可以使用 typedef 来定义一个 block 类型：

  ```objective-c
  typedef 返回值类型 (^声明名称)(参数列表);
  ```
  
  ```objectivec
    typedef int(^MyBlock)(int);
  ```
    然后就可以使用 MyBlock 这个类型来定义变量了：

    ```objectivec
    MyBlock blk_t = ^ (int index) { return index; }
    ```
    与刚才举例的对比
    ```objectivec
    //原来的记述方式
    void func(int (^blk)(int))
    //用了 typedef 定义后的记述方式
    void func(MyBlock blk)

    //原来的记述方式
    int (^func()(int)) 
    //用了 typedef 定义后的记述方式
    MyBlock func()
    ```
***
# Block截取变量
- 截获自动变量：如果在 block 中使用到上文中的局部变量的话，block 会将其保存到自身的内部中，保存的是该变量声明时候的瞬时值,不管下文那个变量怎么变，在 block 调用的时候，用的变量值都是之前截获的那个值。比如下面代码，程序的输出结果为：**10**

  ```objectivec
  int var = 10;
  MyBlock block = ^ { NSLog(@"%d", var); };
  var = 20;
  block();
  ```
**注意**：如果在Block中修改val的值会报错！
  不过不用担心，我们可以用**__block修饰符**来实现在变量block中修改值的需求。

  ```objectivec
  __block int var = 10;
  MyBlock block = ^ {
      var = 20;
      NSLog(@"%d", var);
  };
  block();
  ```
  这样程序输出结果就为**20**

- 对于全局变量、静态全局变量、静态变量在块中可以修改。
- 如果修改的是Objective-C对象，例如NSArray对象，不可以对其赋值但是可以增删元素。并且同样可以通过__block修饰符进行变量修改。
- 不能在block中访问C语言字符数组
  ```objectivec
  const char text[] = "text";
  MyBlock block = ^ {
      NSLog(@"%c", text[2]);
  };
  ```
	这样会造成编译报错，因为block中并**没有实现对C语言字符数组的截获**，虽然变量的类型以及数组的大小都相同，但C语言规范不允许这种赋值，可以将其改成指针形式：

  ```objectivec
  const char *text = "text";
  MyBlock block = ^ {
      NSLog(@"%c", text[2]);
  };
  ```

# Blocks 变量的循环引用以及如何避免

我们知道 Block 会对引用的局部变量进行持有。同样，如果 Block 也会对引用的对象进行持有，从而会导致相互持有，引起循环引用。

```objective-c
/* —————— retainCycleBlcok.m —————— */   
#import <Foundation/Foundation.h>
#import "Person.h"
int main() {
    Person *person = [[Person alloc] init];
    person.blk = ^{
        NSLog(@"%@",person);
    };

    return 0;
}


/* —————— Person.h —————— */ 
#import <Foundation/Foundation.h>

typedef void(^myBlock)(void);

@interface Person : NSObject
@property (nonatomic, copy) myBlock blk;
@end


/* —————— Person.m —————— */ 
#import "Person.h"

@implementation Person    

@end
```

上面 `retainCycleBlcok.m` 中 `main()` 函数的代码会导致一个问题：person 持有成员变量 myBlock blk，而 blk 也同时持有成员变量 person，两者互相引用，永远无法释放。就造成了循环引用问题。

那么，如何来解决这个问题呢？

## ARC 下，通过 __weak 修饰符来消除循环引用

在 ARC 下，可声明附有 __weak 修饰符的变量，并将对象赋值使用。

```objective-c
int main() {
    Person *person = [[Person alloc] init];
    __weak typeof(person) weakPerson = person;

    person.blk = ^{
        NSLog(@"%@",weakPerson);
    };

    return 0;
}
```

这样，通过 __weak，person 持有成员变量 myBlock blk，而 blk 对 person 进行弱引用，从而就消除了循环引用。

## MRC 下，通过 __block 修饰符来消除循环引用

MRC 下，是不支持 **weak 修饰符的。但是我们可以通过** block 来消除循环引用。

```objective-c
int main() {
    Person *person = [[Person alloc] init];
    __block typeof(person) blockPerson = person;

    person.blk = ^{
        NSLog(@"%@", blockPerson);
    };

    return 0;
}
```

通过 __block 引用的 blockPerson，是通过指针的方式来访问 person，而没有对 person 进行强引用，所以不会造成循环引用。