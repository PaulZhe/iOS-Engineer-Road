## 引语
- 用简单的赋值语句将对象赋值给另一个对象时发生的情况：
    ```objective-c
    Object x, y;
    y = x;
    x.name = objectA.name;
    ```
    在这段代码块执行结束后，打印x和y的name属性，会发现它们存储的对象都是```objectA.name```。这是因为这样赋值的结果仅仅是将对象y的地址复制到x中，在赋值操作结束时，两个变量都指向内存中的同一个地址。
***
## copy和mutableCopy方法
- Foundation类实现了名为copy和mutableCopy的方法，可以使用这些方法创建对象的副本，通过实现一个符合<NSCopying>协议（用于制作副本）的方法来完成此任务。如果类必须要区分产生对象的是可变副本还是不可变副本，还需要根据<NSMutableCopying>协议实现一个方法，下面会提到如何实现这个方法。
- 如何让自己的类用 copy 修饰符？
    - 具体步骤：
        1. 需声明该类遵从 NSCopying 协议
        2. 实现 NSCopying 协议。该协议只有一个方法:
            ```objectivec
            - (id)copyWithZone:(NSZone *)zone;
            ```
    - 若想令自己所写的对象具有拷贝功能，则需实现 NSCopying 协议。如果自定义的对象分为可变版本与不可变版本，那么就要同时实现 NSCopying 与 NSMutableCopying 协议。
- 如何重写带 copy 关键字的 setter？
   ```objective-c
    - (void)setName:(NSString *)name {
        _name = [name copy];
    }
   ```

- 产生一个对象的可变副本并不要求被赋值的对象本身是可变的。这种情况同样使用于不可变副本：可以创建可变对象的不可变副本。
***
## 浅复制与深复制
- 下面中的mutableStrArray是由3个可变字符串组成其元素的可变数组。
    ```objectivec
    NSMutableArray *mutableStrArray2 = [mutableStrArray mutableCopy];
    NSMutableString *mStr = mutableStrArray[0];
    [mStr appendString:@"appendStr"];
    ```
    打印两个数组中所有元素会发现mutableStrArray和mutableStrArray2中第一个元素的可变字符串都会加上字符串@"appendStr"。
    1. 原始数组的第一个值为什么会被修改好理解，因为：从集合中获取元素时，得到的是这个元素的一个引用，但并不是一个新副本。所以，修改字符串对象mStr的副作用就是同时改变了mutableStrArray的第一个元素。
    2. 但是为什么我们制作的副本的第一个元素也会改变呢？这与默认的浅复制方式有关。它意味着使用mutableCopy方法复制数组时，在内存中为新的数组对象分配了空间，并且将单个元素复制到新数组中。然而将原始数组中的每个元素复制到新位置并不是指为每个元素制作相应的副本元素，而是仅将引用从一个数组元素复制到另一个数组元素。这样做的最终结果，就是这两个数组中的元素都指向内存中的同一个字符串。这与将一个对象赋值给另一个对象没有区别，就像我们引语中举的例子一样。
    3. 若要为数组中的每个元素创建完全不同的副本，需要执行所谓的深复制。这就意味着要创建数组中的每个对象内容的副本。然而使用Foundation类的copy和mutableCopy方法时，深复制并不是默认执行的。
- 首先来看看复制的两种情况：
    - 对非集合类对象的 copy 与 mutableCopy 操作；
    - 对集合类对象的 copy 与 mutableCopy 操作。
    1. 对非集合类对象的copy与mutableCopy：在非集合类对象中：对不可变对象进行 copy 操作，是指针复制，mutableCopy 操作时内容复制；对可变对象进行 copy 和 mutableCopy 都是内容复制。用代码简单表示如下：
    ```objective-c
    [immutableObject copy] // 浅复制
    [immutableObject mutableCopy] //深复制
    [mutableObject copy] //深复制
    [mutableObject mutableCopy] //深复制
    ```
    比如以下代码都是内容复制，即深拷贝：
    ```objective-c
    NSMutableString *string = [NSMutableString stringWithString:@"origin"];//copy
    NSString *stringCopy = [string copy];
    ```
    2. 集合类对象的copy与mutableCopy：在集合类对象中，对 immutable 对象进行 copy，是指针复制， mutableCopy 是内容复制；对 mutable 对象进行 copy 和 mutableCopy 都是内容复制。但是：集合对象的内容复制仅限于对象本身，对象元素仍然是指针复制。用代码简单表示如下：
    ```objective-c
    [immutableObject copy] // 浅复制
    [immutableObject mutableCopy] //单层深复制
    [mutableObject copy] //单层深复制
    [mutableObject mutableCopy] //单层深复制
    ```
    参考链接： [iOS 集合的深复制与浅复制](https://www.zybuluo.com/MicroCai/note/50592)
- 集合的深复制(deep copy)：集合的深复制有两种方法。
  1. 可以用 initWithArray:copyItems: 将第二个参数设置为YES即可深复制，如
        ```objective-c
        NSDictionary shallowCopyDict = [[NSDictionary alloc] initWithDictionary:someDictionary copyItems:YES];
        ​```objectivec
     如果你用这种方法深复制，集合里的每个对象都会收到 copyWithZone: 消息。如果集合里的对象遵循 NSCopying 协议，那么对象就会被深复制到新的集合。如果对象没有遵循 NSCopying 协议，而尝试用这种方法进行深复制，会在运行时出错。copyWithZone: 这种拷贝方式只能够提供一层内存拷贝(one-level-deep copy)，而非真正的深复制。
     ```
    2. 第二个方法是将集合进行归档(archive)，然后解档(unarchive)，如：
        ```objective-c
        NSArray *trueDeepCopyArray = [NSKeyedUnarchiver unarchiveObjectWithData:[NSKeyedArchiver archivedDataWithRootObject:oldArray]];
        ```
##  怎么用 copy 关键字？
- 用途：
    1. NSString、NSArray、NSDictionary 等等经常使用copy关键字，是因为他们有对应的可变类型：NSMutableString、NSMutableArray、NSMutableDictionary；
    2. block 也经常使用 copy 关键字，在 ARC 中写不写都行：对于 block 使用 copy 还是 strong 效果是一样的，但写上 copy 也无伤大雅，还能时刻提醒我们：编译器自动对 block 进行了 copy 操作。具体原因见官方文档。
- 如果属性声明中指定了copy特性，合成方法会使用类的copy方法，**这里注意：属性并没有mutableCopy特性。即使是可变的实例变量，也是使用copy特性，正如方法 ```copyWithZone:```的执行结果。所以，按照约定会生成一个对象的不可变副本。**
- 这里就引出了一个思考题：这个写法会出什么问题： ```@property (noatomic, copy) NSMutableArray *array;```
    - 添加,删除,修改数组内的元素的时候,程序会因为找不到对应的方法而崩溃.因为 copy 就是复制一个不可变 NSArray 的对象
- 用@property声明的NSString（或NSArray，NSDictionary）经常使用copy关键字，为什么？如果改用strong关键字，可能造成什么问题？
    1. 因为父类指针可以指向子类对象,使用 copy 的目的是为了让本对象的属性不受外界影响,使用 copy 无论给我传入是一个可变对象还是不可对象,我本身持有的就是一个不可变的副本.
    2. 如果我们使用是 strong ,那么这个属性就有可能指向一个可变对象,如果这个可变对象在外部被修改了,那么会影响该属性.