[TOC]
# 引言

- 在ios中，事件UIEvent类来表示，当一个事件发生时，系统会搜集的相关事件信息，创建一个UIEvent对象，最后将该事件转发给应用程序对象(UIApplication)。

- 在 UIKit 中，UIApplication、UIView、UIViewController 这几个类都是直接继承自 UIResponder 类。这些对象都有响应事件的能力，因此也被称为响应对象，或者响应者。

- 一个UIResponder类为那些需要响应并处理事件的对象定义了一组接口。
  这些事件主要分为三类：触摸事件，加速计事件(运动事件，比如重力感应和摇一摇等)以及远程遥控事件。下面是官方的一张图片：

  ![img](https://images2015.cnblogs.com/blog/277577/201509/277577-20150925160114115-1117691710.png)

- **当用户通过以上方式触发一个事件时，会将相应的事件对象添加到UIApplication的事件队列中。UIApplication会循环的从队列中拿出第一个事件来处理。首先将该事件分发给UIApplication 的主窗口对象(KeyWindow)，然后由主窗口决定如何将事件交给最合适的响应者(UIResponder)来处理取决于事件的类型**。这里主要分两种情况：

  1. 触摸事件：UIApplication通过一个触摸检测来决定最合适来处理该事件的响应者，一般情况下，这个响应者是UIView对象。

  2. 加速计事件或远程遥控事件：UIApplication寻找UIWindow中的第一响应者。找到第一响应者(The First Responder)后，会将该事件对象派发给该响应者以便处理。

- 下面主要讲解 `Touch Events(触摸事件)` `Touch Events`事件的整个过程可以分为 `传递`和`响应` 2 个阶段，
  
  - 传递： 是当我们触摸屏幕时，为我们找出最适合的 `View`，
  - 响应： 当我们找出最适合的 `View` 后，此时只是找到了最合适的 `View`，但未必 此 `View` 可以响应此事件，所以需要继续找出能响应此事件的 `View`。

# 触摸事件

## 传递过程

- 在触摸事件中上面提到的“由主窗口决定如何将事件交给最合适的响应者”的过程是如何实现的呢？首先我们需要明确一个UIView对象能够接收触摸事件至少要保证以下三个条件：

  1. userInteractionEnabled属性为YES，该属性表示允许控件同用户交互。
	2. Hidden属性为NO。控件都看不见，自然不存在触摸
  3. opacity属性值0 ～0.01。
	4. 触摸点在这个UIView的范围内。

- 每当手指接触屏幕，操作系统会把事件传递给当前的 `App`， 在 `UIApplication`接收到手指的事件之后，就会去调用`UIWindow的hitTest:withEvent:`，看看当前点击的点是不是在window内，如果是则继续依次调用其 subView的`hitTest:withEvent:`方法，直到找到最后需要的view。调用结束并且hit-test view确定之后，便可以确定最合适的 View。

- 引用几张图来说明一下：

  ![img](https://user-gold-cdn.xitu.io/2018/8/1/164f4056fb4e4614?w=1258&h=601&f=png&s=78449)

  ![img](https://user-gold-cdn.xitu.io/2018/8/1/164f4056fb573c0d?w=1258&h=601&f=png&s=119853)

- 下面是 `- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event` 方法的内部实现
	```objective-c
	- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event
  {
      if (self.hidden || !self.userInteractionEnabled || self.alpha < 0.01 || ![self pointInside:point withEvent:event] || ![self _isAnimatedUserInteractionEnabled]) {
          return nil;
      } else {
          for (UIView *subview in [self.subviews reverseObjectEnumerator]) {
              UIView *hitView = [subview hitTest:[subview convertPoint:point fromView:self] withEvent:event];
              if (hitView) {
                  return hitView;
              }
          }
          return self;
      }
  }
  
  ```
  
  ***注意：这里遍历子视图时是逆序的，即遍历 subview 时，是从上往下顺序遍历的，即 view.subviews 的 lastObject 到 firstObject 的顺序，找到合适的响应者view，即停止遍历.***

## 响应过程

- 当我们知道最合适的 View 后，事件会 由上向下【子view -> 父view，控制器view -> 控制器】来找出合适响应事件的 View，来响应相关的事件。如果当前的 View 有添加手势，那么直接响应相应的事件，不会继续向下寻找了，如果没有手势事件，那么会看其是否实现了如下的方法：

  ```objective-c
  - (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event;//开始触摸
  - (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event;//手指移动
  - (void)touchesEnded:(NSSet *)touches withEvent:(UIEvent *)event;//结束触摸
  - (void)touchesCancelled:(NSSet *)touches withEvent:(UIEvent *)event;://触摸终端
  ```

- 如果有实现那么就由此 View 响应，如果没有实现（即不能处理当前事件），那么事件将会沿着响应者链(Responder Chain)进行传递，直到遇到能处理该事件的响应者(Responsder Object)。

  - 这里我们可以做一个简单的验证，**在默认情况下 UIView 是不响应事件的，UIControl 就算没有添加手势一样的会由他来响应**， 这里可以使用 runtime查看 UIView 和 UIControl 的方法列表，  UIView 没有实现如上的 `touchesBegan`方法，而 `UIControl` 是实现了如上的相关方法，所以验证了刚才的 UIView 不响应，和 UIControl 的响应。

- 在响应方法内部，我们也可以将这个触摸事件继续传递给父控件的对应方法处理。

  ```objective-c
  - (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
      [super touchesBegan:touches withEvent:event];//让下一个响应者可以有机会继续处理
  }
  - (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
      [super touchesBegan:touches withEvent:event];
  }
  - (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
      [super touchesBegan:touches withEvent:event];
  }
  - (void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
      [super touchesBegan:touches withEvent:event];
  }
  ```

### 响应者链

- 这里提到了响应者链的概念：响应者链条是由多个响应者对象（即继承于UIResponder的对象）连接起来的链条，我们知道UIResponder类里有个属性：

  ```objective-c
  @property(nonatomic, readonly, nullable) UIResponder *nextResponder;
  ```

  这条链是由 nextResponder 指针连接起来的

- 响应者链的事件传递机制（即触摸事件的响应过程）如下：

  1. 如果 view 是一个 viewController 的 root view，nextResponder 是这个 viewController.

     如果 view 不是 viewController 的 root view，nextResponder 则是这个 view 的 superview

  2. 如果 viewController 的 view 是 window 的 root view, viewController 的 nextResponder 是这个 window

     如果 view controller 是被其他 view controller presented调起来的，那么 view controller 的 nextResponder 就是发起调起的那个 view controller

  3. window 的 nextResponder 是 UIApplication 对象.

  4. 如果UIApplication也不能处理该事件或消息，则将其丢弃

# 第一响应者 (The First Responder)

- 什么是第一响应者？简单的讲，第一响应者是一个UIWindow对象接收到一个事件后，第一个来响应的该事件的对象。注意：这个第一响应者与之前讨论的触摸检测到的第一个响应的UIView并不是一个概念。第一响应者一般情况下用于处理非触摸事件（手机摇晃、耳机线控的远程空间）或非本窗口的触摸事件（键盘触摸事件）。

- 第一响应对象和其他响应对象之间有什么区别？对于普通的触摸事件没什么区别。就算我把一个按钮设置成第一响应对象，当我点击其他按钮时，还是会响应其他按钮，而不会优先响应第一响应对象。第一响应对象的区别在于负责处理那些和屏幕位置无关的事件，例如摇动。

- 苹果官方文档的说法是：第一响应对象是窗口中，应用程序认为最适合处理事件的对象。一个班只能有一个班长，应用的响应对象中，只能有一个响应对象成为第一响应对象。

- 程序员负责告诉系统哪个对象可以成为第一响应者(canBecomeFirstResponder)，如果方法canBecomeFirstResponder返回YES，这个响应者对象才有资格称为第一响应者。有资格并不代表一定可以成为第一响应者，还差becomeFirstResponder正式成为第一响应者。

- 值得注意的是，一个UIWindow对象在某一时刻只能有一个响应者对象可以成为第一响应者。我们可以通过isFirstResponder来判断某一个对象是否为第一响应者。

- 在当上第一响应对象时，不同对象可能会有一些特殊的表现。例如UITextField当上的时候，就会调出一块小键盘。

  第一响应对象也有可能被辞退。发送一个resignFirstResponder，就可以劝退。
***
**参考博客：**
1. [解析iOS开发中的FirstResponder第一响应对象](https://www.cnblogs.com/On1Key/p/5306747.html)
2. [ios中的事件处理、响应者链条以及第一响应者](https://www.cnblogs.com/forwk/p/4843159.html)
3. [浅谈 iOS 事件的传递和响应过程](https://www.e-learn.cn/content/qita/1052137)