## 前言

主要记录一下`respondsToSelector`方法与`instancesRespondToSelector`方法的区别，对于很多人来说，平时使用得也比较多，但是可能会对这两个方法的使用弄混淆，希望写在这里对一些同学可以起到一些帮助。

## 结论

### respondsToSelector方法

- 本身是一个`实例方法` 

```objectivec
- (BOOL)respondsToSelector:(SEL)aSelector;
```

- 用于`class`调用(class本身也是一个对象)

**判断class是否实现了相应的类方法。**

- 用于`object`调用

**判断实例对象是否实现了相应的实例方法。**

### instancesRespondToSelector方法

- 本身是一个`类方法` 

```objectivec
+ (BOOL)instancesRespondToSelector:(SEL)aSelector;
```

- 只能用`类`来调用，不能用`实例对象`来调用

**判断该类的实例对象是否实现了相应的实例方法。**



**[原文链接](https://www.jianshu.com/p/29b9c6abfc1b)**