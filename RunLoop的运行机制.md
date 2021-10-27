一个线程一次只能执行一个任务，执行完之后线程就会退出，如果我们需要机制，让线程能随时处理

----
- 如何管理事件和消息
- 如何让线程在没有处理消息时休眠，避免无谓的资源占用
- 如何在有消息到来时立即被唤醒

RunLoop本质是一个对象，这个对象管理了其需要处理的事件和消息，并提供了入口函数来执行event loop的逻辑，线程执行这个函数后，就会一直处理这个函数内部： 接受消息-> 等待-> 执行的循环中。

OSX/iOS 系统中，提供了两个这样的对象：NSRunLoop 和 CFRunLoopRef。
- CFRunLoopRef 是在 CoreFoundation 框架内的，它提供了纯 C 函数的 API，所有这些 API 都是线程安全的。
- NSRunLoop 是基于 CFRunLoopRef 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的

### RunLoop与线程的关系
- iOS开发中遇到两个线程对象：pthread_t和NSThread,他们是一一对应的。
- 苹果不允许直接创建RunLoop,它只提供了两个自动获取的函数：CFRunLoopGetMain()和CFRunLoopGetCurrent()
- 线程和RunLoop是一一对应的，其关系是保存在一个全局的Dict中。线程刚创建时并没有RunLoop，它的创建是发生在第一次获取时，RunLoop的销毁时发生在线程结束时。只能在一个线程的内部获取RunLoop

RunLoop的特性
- 主线程的RunLoop在应用启动的时候就会自动创建
- 其他线程则需要在该线程下自己启动
- 不能直接创建RunLoop
- RunLoop并不是线程安全的，所以需要避免在其他线程上调用当前线程的RunLoop
- RunLoop负责管理autorelease pools
- RunLoop负责处理消息事件，即输入源事件和计时器事件

一个RunLoop可以包含多个Mode，每个Mode有包含多个Source、Timer、Observer。每次调用RunLoop主入口函数时，只能指定一个Mode，这个Mode被称作CurrentMode。如果需要切换Mode，需要退出本次循环，再重新指定一个Mode进入。

Source、Timer、Observer 被统称为 mode item，一个 item 可以被同时加入多个 mode。但一个 item 被重复加入同一个 mode 时是不会有效果的。如果一个 mode 中一个 item 都没有，则 RunLoop 会直接退出，不进入循环。

- CFRunLoopSourceRef是事件产生的地方，有两个版本，Source0和Source1,Source0只包含了一个回调（函数指针），它并不能主动触发事件。使用时，你需要先调用CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。
- CFRunLoopTimerRef 是基于时间的触发器，它和 NSTimer 是toll-free bridged的，可以混用。其包含一个时间长度和一个回调（函数指针）。当其加入到 RunLoop 时，RunLoop会注册对应的时间点，当时间点到时，RunLoop会被唤醒以执行那个回调。
- CFRunLoopObserverRef 是观察者，每个 Observer 都包含了一个回调（函数指针），当 RunLoop 的状态发生变化时，观察者就能通过回调接受到这个变化。

实际上 RunLoop 其内部是一个 do-while 循环。RunLoop的核心就是一个MachMessage的调用过程，RunLoop调用这个函数去接收消息，如果没有别人发送port消息过来，RunLoop会进入休眠状态。内核会将线程置于等待状态，这个时候whild循环是停止的，相比于一直运行的while循环，会很节省CPU资源，进而达到省电的目的

在Mach中，所有的组件都是一个对象。进程、线程、虚拟内存都是对象。Mach对象之间不能直接调用，对象间的通讯是通过消息机制实现的

- 为什么引入Runloop机制，有什么作用或者好处？
- 引入Runloop机制的目的是利用RunLoop机制的特点实现整体省电的效果，并且让系统和应用可以流畅的运行，提高响应速度，达到极致的用户体验。
- 为什么省电？
- 一个app流畅与否的决定性因素是主线程的阻塞率，在iOS系统中runloop每秒执行60次，理论上主线程runloop达到55帧以上的刷新频率用户就感觉不到卡顿
- Mode机制，同一时间只执行一个Mode内的Source或者Timer，比如拖动的时候只指定拖动Mode，其他Mode 如Default Mode中的源不会被执行，确保了高速滑动的时候不会有其他逻辑阻碍主线程刷新。
- Runloop做的是管理视图刷新频率，防止重复运算。由于视图更新必须在主线程，视图的重布局和重绘都会占用主线程的执行时间，一次Runloop循环只执行一次可以最大限度的防止重复运算导致的计算浪费。
- 管理核心动画。核心动画有三个树，其中render tree 是私有的，应用开发无法访问到。render tree在专用的render server 进程中执行，是真正用来渲染动画的地方，线程优先级高于主线程。所以即使app主线程阻塞，也不会影响到动画的绘制工作。既节省了主线程的计算资源，又使动画可以流畅的执行。
- 支持异步方法调用，将耗时操作分发到子线程中进行。RunLoop是performSelector的基础设施。我们使用 performSelector:onThread: 或者 performSelecter:afterDelay: 时，实际上系统会创建一个Timer并添加到当前线程的RunLoop中

管理自动释放池
- App启动后，主线程RunLoop中会注册Observer，分别在RunLoopEntry、BeforeWaiting和Exit时调用。回调都是_wrapRunLoopWithAutoreleasePoolHandler()。
- RunLoopEntry Observer 在RunLoop进入的时候会被触发，在_wrapRunLoopWithAutoreleasePoolHandler()函数中调用_objc_autoreleasePoolPush()创建自动释放池。order是- 2^31，优先级最高，确保在所有回调之前创建自动释放池。
- RunLoopBeforWaiting Observer 在RunLoop进入休眠之前被触发，同样在_wrapRunLoopWithAutoreleasePoolHandler()函数中处理，但是调用的方法不同，分别调用_objc_autoreleasePoolPop()和_objc_autoreleasePoolPush()方法。释放旧的自动释放池，同时创建新的自动释放池。
- RunLoopExit Observer 在RunLoop退出的时候被触发。调用_objc_autoreleasePoolPop()释放自动释放池。

管理视图更新

- 这个视图刷新的机制就需要RunLoop去支持。RunLoop Observer会在即将进入休眠 BeforeWaiting 和 退出 Exit 的时候调用CFRunLoopObservermPv()函数，这个函数会遍历所有有标记的View和Layer，执行真正的重新布局和重新绘制方法。达到刷新视图界面的目的。

管理Core Animation

- model tree是我们可以直接操作的tree，当修改CALayer的时候，CALayer的属性值会修改model tree。
- presentation tree 是layer在屏幕中的真实位置也是一个CALayer对象。可以通过view.layer.presentationLayer 获得。presentation是只读的
- render tree 是私有的，应用开发无法访问到。render tree在专用的render server 进程中执行，是真正用来渲染动画的地方，线程优先级高于主线程。所以即使app主线程阻塞，也不会影响到动画的绘制工作。无论隐式还是显式动画都是在当前线程的RunLoop结束后提交到render tree。因为 commit transaction 操作是从app进程到render server 进程是IPC，会有进程间通讯开销，所以官方不推荐我们手动 commit transaction