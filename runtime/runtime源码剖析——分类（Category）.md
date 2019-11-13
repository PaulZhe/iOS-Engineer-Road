#  开发时用 Category 做了哪些事？

- 主要作用：为已经存在的类添加方法
- 声明私有方法
- 分解体积庞大的类文件（优点：减少单个文件体积，功能模块化，多个开发者共同完成一个类）
- 模拟多继承
- 把Framework的私有方法公开化

# category和extension

`extension`看起来很像一个匿名的`category`，但是`extension`和有名字的`category`几乎完全是两个东西：

- `extension`在编译器决议，它就是类的一部分，在编译期和头文件里的`@interface`以及实现文件的`@implement`一起形成一个完整的类，伴随类的生命周期一起存亡，所以无法为系统的类比如`NSString`添加`extension`
- `category`则是在运行期决议的。`category`是无法添加实例变量的（因为在运行期，对象的内存布局已经确定，如果添加实例变量就会破坏类的内部布局，这对编译型语言来说是灾难性的）。

## Non Fragile ivars

这个特性的介绍可以看这篇文章：[Objective-C类成员变量深度剖析](http://quotation.github.io/objc/2015/05/21/objc-runtime-ivar-access.html)，总结一下：

> 1) 用旧版OSX SDK编译的MyObject类成员变量布局是这样的，MyObject的成员变量依次排列在基类NSObject的成员后面。
>
> ![旧版本SDK的成员变量布局](http://quotation.github.io/images/20150521/nf1.png)
>
> 2) 当苹果发布新版本OSX SDK后，NSObject增加了两个成员变量。如果没有Non Fragile ivars特性，我们的代码将无法正常运行，因为MyObject类成员变量布局在编译时已经确定，有两个成员变量和基类的内存区域重叠了。此时，我们只能重新编译MyObject代码，程序才能在新版本系统上运行。如果更悲催一点，MyObject类是来自第三方提供的静态库，我们就只能眼巴巴等着库作者更新版本了。
>
> ![新版本SDK的成员变量布局](http://quotation.github.io/images/20150521/nf2.png)
>
> 3) Non Fragile ivars特性出场了。在程序启动后，runtime加载MyObject类的时候，通过计算基类的大小，runtime动态调整了MyObject类成员变量布局，把MyObject成员变量的位置向后移动8个字节。于是我们的程序无需编译，就能在新版本系统上运行。
>
> ![Runtime调整后的布局](http://quotation.github.io/images/20150521/nf3.png)

这里提到的这个特性的原因是刚才提到的那个问题：`category`为什么无法添加实例变量？

这个问题的答案与`Non Fragile ivars`无关，但有容易混淆的地方，故放在一起讨论。那为什么runtime允许动态添加方法和属性，而不会引发问题呢？

因为对象在内存中的排布可以看成一个结构体，该结构体的大小并不能动态变化。所以无法在运行时动态给对象增加成员变量。但方法定义是在`objc_class`（类对象）中管理的，不管如何增删方法，都不影响对象的内存布局，已经创建出的类实例仍然可正常使用。

# 分类的特点（和扩展的区别）

- 运行时决议
- 可以为系统类添加分类

# 分类都可以添加哪些内容

- 实例方法

- 类方法

- 协议

- 属性（无法合成相应的实例变量，并没有在分类中添加实例变量（）通过关联对象添加）

  - 引申问题：能否给分类添加“成员变量”？

    在下文详细解答。

> 我们来总结一下 **Category 的本质**：
>
> Category 的本质就是 `_category_t 结构体` 类型，其中包含了以下几部分：
>
> 1. `_method_list_t` 类型的『对象方法列表结构体』；
> 2. `_method_list_t` 类型的『类方法列表结构体』；
> 3. `_protocol_list_t` 类型的『协议列表结构体』；
> 4. `_property_list_t` 类型的『属性列表结构体』。
>
> `_category_t 结构体` 中不包含 `_ivar_list_t` 类型，也就是不包含『成员变量结构体』。

```cpp
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always pre sent on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```

-   ![](https://tva1.sinaimg.cn/large/006y8mN6ly1g7orhrhn2uj312p0kwq3w.jpg)
- 在`remethodizeClass`方法中是倒序遍历分类列表的，即最先访问最后编译的分类。假如分类有多个的话，其中有同名方法，那么最后编译的方法会生效，因为前面的会被覆盖掉。
- 虽说被“覆盖”掉了，但实际还是在方法列表中的，如下图，只是因为最后被编译的方法存在方法列表的前面，在查找方法的时候查到相应的方法就会返回，注意：**这里还有一个点，如果分类中有和宿主类同名的方法时，分类中的方法会被优先实现。**![](https://tva1.sinaimg.cn/large/006y8mN6gy1g7osw919dbj30pa0gaabq.jpg)
- ![](https://tva1.sinaimg.cn/large/006y8mN6ly1g7oskxx9ulj30ul0e875y.jpg)
- 刚遗留了一个问题：能否给分类添加“成员变量”？
  - 不能添加成员变量，但可以通过关联对象达到成员变量的效果，我们不能在分类的声明或者定义的时候为分类添加成员变量，但是可以通过**关联对象**的技术来为分类添加成员变量，来达到给分类添加成员变量的效果。
  - 关联对象的方法：![](https://tva1.sinaimg.cn/large/006y8mN6gy1g7ote6vtb1j312g0d8dhe.jpg)
  - 关联对象由`AssociationsManager`管理并在`AssociationsHashMap`存储。
  - 所有对象的关联内容都在同一个全局容器中。

# 总结

- 分类添加的方法可以“覆盖”原类方法
- 同名分类方法谁能生效取决于编译顺序
- 名字相同的分类会引起编译报错