[TOC]

# 工厂方法模式

## 定义

定义一个用于创建对象的接口，让子类决定实例化哪一个类。工厂方法使一个类的实例化延迟到其子类。

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8vn4fgjjlj30dl08vdh3.jpg)

## 优点

- 良好的封装性，代码结构清晰，降低模块间的耦合。
- 扩展性很优秀。在增加产品类的情况下，只要适当地修改具体的工厂类或扩展一个工厂类，就可以完成“拥抱变化”。
- 可以屏蔽产品类。产品类的实现如何变化，调用者都不需要关心，它只需要关心产品的接口，只要接口保持不变，系统中的上层模块就不要发生变化。
- 工厂方法模式是典型的解耦框架。高层模块只需要知道产品的抽象类，其他的实现类都不用关心，符合迪米特法则 ，我不需要的就不要去交流 ;也符合依赖倒置原则，只依赖产品类的抽象:当然也符合里氏替换原则，使用产品子类替换产品父类。

## 工厂方法模式的扩展

1. 缩小为简单工厂模式

   实际开发中会考虑这么一个问题：一个模块仅需要一个工厂类，没有必要为其设置抽象类。

   以《设计模式之禅》中女娃造人的例子来看，原本工厂方法的 UML 类图如下图所示：

   ![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8vndzi4q7j30ra0g8acr.jpg)

   简单工厂模式的做法的 UML 类图如下图所示：

   ![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8vnk6lhumj30o20gbn1r.jpg)

   这种扩展中我们去除了 AbstractHumanFactory 抽象类，同时把 createHuman 方法设置为类方法。去掉了抽象基类，同时引起了调用者的变化。

   ### 在项目中应用
   
   - 我在自己一个原生 APP 内嵌 H5 网页开发的设计中用到了简单工厂模式，它的大致需求就是 APP 界面是 H5网页，由 Web 端调取 iOS 原生的各种权限，其中功能被模块化，在刚开始写好以后发现 ViewController 太过臃肿，且实现部分暴露给了客户端，这违背了设计原则中的迪米特法则。于是我思考之后将代码重构，用到了简单工厂模式。
   
   - ![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8wjqt2rf9j30s00b6wek.jpg)
   
     上图是我自己画的对项目架构的设计，将原来代码中冗余部分都封装到了各自的类中，继承于`JSBBaseModule`，有个共有方法`- (id)performModuleMethodWithDictionary:(NSDictionary *)messageDictionary`，替代原先写在 ViewController 中的方法。
   
   - 工厂生产出的具体哪个类由运行时 Web 端发过来的字段决定，用`JSBBaseModule`接收，并执行刚才说的共有方法，这里就又遵循了设计原则中的里氏替换原则，
   
   - 至于如何根据字段选择生产的类最早采用的方案是用万能的`if else`(￣◇￣;)，方案以前的一片博客中有写：[OC枚举类型和字符串互转结合.pch文件的使用](https://blog.csdn.net/qq_42347755/article/details/97899337)。重构后采用了`plist`配置文件结合反射机制（反射机制了解可参照[刘小壮写的iOS反射机制](https://www.jianshu.com/p/5bbde2480680)）来实现，代码如下，名为`ModuleName`的`plist`文件存的是以**Web 端发过来的字段为键，真正要调用的方法名为值**的键值对：
   
     ```objectivec
     + (JSBBaseModule *)createConcreteModule:(NSString *)moduleName {
         NSString *path = [[NSBundle mainBundle] pathForResource:@"ModuleName" ofType:@"plist"];
         NSDictionary *dic = [[NSDictionary alloc] initWithContentsOfFile:path];
         NSString *value = [dic valueForKey:moduleName];
         
         SEL createModule = NSSelectorFromString(value);
         JSBBaseModule *module;
         if ([self respondsToSelector:createModule]) {
             module = [self performSelector:createModule];
         }
         return module;
     }
     ```
   
2. 升级为多个工厂类

   当我们在做一个比较复杂的项目时，经常会遇到初始化一个对象很耗费精力的情况，所有的产品类都放到一个工厂方法中进行初始化会使代码结构不清晰。

   考虑到需要结构清晰，我们就为每个产品定义一个创造者，然后由调用者自己去选择与哪个工厂方法关联。我们还是以女娲造人为例，每个人种都有一个固定的八卦炉，分别造出黑色人种、白色人种、黄色人种，修改后的类图如图 8-4所示。 

   ![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8wpbixrsdj30io0fegoj.jpg)

   每个人种(具体的产品类 ) 都对应了一个创建者，每个创建者都独立负责创建对应的产品
   对象，非常符合单一职责原则 。	

# 抽象工厂模式

## 定义

为创建一组相关或相互依赖的对象提供一个接口，而且无需指定它们的具体类。

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8wpnuchpgj30dp0bh0to.jpg)

抽象工厂模式是工厂方怯模式的升级版本，在有多个业务品种、业务分类时，通过抽象工厂模式产生需要的对象是一种非常好的解决方式。

我们来看看抽象工厂的通用源代码，首先有两个互相影响的产品线(也叫做产品族)，例如制造汽车的左侧门和右侧门，这两个应该是数量相等的一一两个对象之间的约束，每个型号的车门都是不一样的，这是产品等级结约束的，我们先看看两个产品族的类图，如图 9-4所示。

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8wpqs5gd3j30fz0i2tal.jpg)

## 优点

- 封装性 ，每个产品的实现类不是高层模块要关心的，它要关心的是什么?是接口，是抽
  象，它不关心对象是如何创建出来，这由谁负责呢?工厂类，只要知道工厂类是谁，我
  就能创建出一个需要的对象。
- 产品族内的约束为非公开状态，具体的产品族内的约束是在工厂内实现的。

## 缺点

抽象工厂模式的最大缺点就是产品族扩展非常困难，为什么这么说呢?我们以通用代码为例，如果要增加一个产品C，也就是说产品家族由原来的2个增加到3个，看看我们的程序有多大改动吧! 抽象类AbstractCreator要增加一个方法createProductC()，然后两个实现类都要修改，想想看，这严重违反了开闭原则，而且我们一直说明抽象类和接口是一个契约。改变契约，所有与契约有关系的代码都要修改，那么这段代码叫什么?叫“有毒代码”，一一只要与这段代码有关系，就可能产生侵害的危险!

# 工厂方法与抽象工厂比较

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8wqbkisbdj30qg08075s.jpg)

