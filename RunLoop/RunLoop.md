- ![](https://i.loli.net/2019/09/26/6JCurBFjTXcizeN.jpg)

- CFRunLoopSource：
	-  Source 是 RunLoop 的数据源抽象类（protocol）  
	- RunLoop 定义了两个 Version 的 Source：
	  1. Source0: 处理 APP 内部事件、APP 自己负责管理（触发），如UIEvent、CFSocket。
	  2.    Source1:由RunLoop和内核管理，Mach port（NSPort是对Machport的上层封装，其功能类似于进程通信中的pipe）驱动，如CFMachPort、CFMessagePort
	- 如有需要，可从中选择一种来实现自己的Source（基本不会发生）。
	
- UIKit  通过 RunLoopObserver 在RunLoop 两次Sleep间对 Autorelease 进行 Pop 和 Push ，将这次 Loop 中产生的 Autorelease 对象释放

- RunLoop 在同一时间只能且必须在一种特定 Mode 下Run

- 更换 Mode 时，需停止当前 Loop，然后重启新 Loop

- Mode 是iOS App 滑动流畅的关键

- 可以定制自己的 Mode

- 几种 Mode ：
	1. NSDefaultRunLoopMode  默认状态，空闲状态
	2. UITrackingRunLoopMode  滑动ScrollerView时
	3. UIInitialiazationRunLoopMode  私有，App启动时
	4. NSRunLoopCommonModes Mode集合，包含前两种Mode
	
- 滑动屏幕时 Mode 的变化：NSDefaultRunLoopMode -> UITrackingRunLoopMode  -> NSDefaultRunLoopMode

- RunLoop 实际上就是一个对象，这个对象管理了其需要处理的事件和消息，并提供了一个入口函数来执行上面 Event Loop 的逻辑。线程执行了这个函数后，就会一直处于这个函数内部 “接受消息->等待->处理” 的循环中，直到这个循环结束（比如传入 quit 的消息），函数返回。

- 线程和 RunLoop 之间是一一对应的，其关系是保存在一个全局的 Dictionary 里。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）。

- 一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。

  ![RunLoop_0](https://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_0.png)