date: 2016-01-27 18:15:50
author: [raven.zhang]
title: ASDK基本原理
tags: [iOS]
---

AsyncDisplayKit是Facebook开源的一套用于提高iOS界面流畅的异步绘制UI的框架。其核心思想是对CPU以及GPU的优化，CPU方面则是在将一些高消耗运算放在后台线程中进行，尽量减轻主线程负担；GPU方面则是将每一帧的内容渲染成一张texture，这个工作可以在后台线程中完成。

AsyncDisplayKit的基本单元是node，Node是UIView以及对应Layer的抽象层。与UIView的最大区别在于，Node是线程安全的，并可以设置对应的Node内部层次后在后台线程中运行，而UIView只能将大量的耗时操作全都放在主线程上进行，下面是AsyncDisplayKit和UIKit的一些类对应关系。

ASDisplayNode. Counterpart to UIView — subclass to make custom nodes.
ASControlNode. Analogous to UIControl — subclass to make buttons.
ASImageNode. Like UIImageView — decodes images asynchronously.
ASTextNode. Like UITextView — built on TextKit with full-featured rich text support.
ASTableView and ASCollectionView. UITableView and UICollectionView subclasses that support nodes.

ASDisplayNode是线程安全的，它可以在后台线程创建和修改。Node刚创建时，并不会在内部创建UIView和CALayer，直到第一次在主线程访问view或者layer属性时，它才会在内部生成对应的对象。当它的属性（比如frame/transform）改变后，它并不会立刻同步到其持有的view或layer去，而是把被改变的属性保存到内部的一个中间变量，稍后在需要时，再通过某个机制一次性设置到内部的view或layer。实际的绘制代码在:ASDisplayNode和ASDisplayNode+AsyncDisplay.mm。

从主线程中剥离出来的耗时运算比较重要的就是图像、布局还有绘制。
图像（解压与处理）：一般UIImage对其内部图像格式的解压发生在即将将图片交给GPU渲染的时候。从API上来看，一般我们调用[UIImage drawInRect]函数或者将UIView可见（放置于window上）的时候，UIImage会将其内部图像格式如PNG，JPEG进行解压。AsyncDisplayKit就是采用后面那个方式将UIView预先放置入一个frame为空得workingview中以达到将view内部的image进行预解压的目的。再说图像的处理。一般图像需要一些blend运算或者图像需要strech或crop，这些工作其实可以留在GPU中进行快速计算，但因为UIKit并没有提供此类的API，所以我们一般都是通过CoreGraphic来做，但CoreGraphic是CPU drawing比较费时。AsyncDisplayKit将这些图像处理放在工作线程中来处理，虽然也是CPU drawing，但不会影响到UI得顺滑响应。

布局：以AsyncDisplayKit的ASTableView为例,TableView的UI布局需要计算每行的高度，然后计算行内元素的布局，将行插入到TableView中，同时TableView又是scrollview，需要上下滑动。一旦行的生成和渲染比较慢，必然影响到滑动时的流畅体验。在这个过程中只有将行插入到TableView中需要在UI线程中执行。AsyncDisplayKit在子线程中分批次计算行（row）以及其子元素的布局，计算好后通过dispatch_group_notify通知UI线程将row插入到view中。AsyncDisplayKit有一个比较细腻的方式是考虑到设备的CPU核数，根据核数来分批次计算row的大小。

绘制：AsyncDisplayKit另一个强大之处在于将UI CALayer的backing image的绘制放入到工作线程。我们知道UIView的真正展现内容在于其CALayer的contents属性，而contents属性对应一个Backing image，我们可以将其理解成一个内存位图。默认情况下UIView一旦需要展现，其会自动创建一个Backing image，但我们也可以通过CALayer的delegate方式来定制这个Backing image。AsyncDisplayKit就是通过CALayer的delegate控制backing image的生成，并且通过Core Graphic的方式在工作线程上将View以及其子节点绘制到backing image上，这些绘制工作会根据UIView的层次构建一个绘制数组统一执行，绘制好之后在UI线程上将这个backing image传给CALayer的contents，最后将CALayer的渲染树交给GPU进行渲染。虽然这个过程中主要依赖于CoreGraphic来进行绘制，但因为都在后台，而且绘制以组的方式执行减少了graphic context的切换，对于UI性能和顺滑性没有什么影响。而且他通过transaction的方式管理dispatch_group之间的关系，并且只有在UI线程处于idle状态或将退出时才将transaction commit并将backing image赋给CALayer的contents。除了通过CAlayer的backing image绘制，AsyncDisplayKit还提供UIView的drawRect绘制以及UIView的rasterize。两者都会使用offscreen drawing，但后者会将UIView以及所有子节点都绘制在一个backing image上。

核心技术：Runloop 任务分发
iOS 的显示系统是由 VSync 信号驱动的，VSync 信号由硬件时钟生成，每秒钟发出 60 次（这个值取决设备硬件，比如 iPhone 真机上通常是 59.97）。iOS 图形服务接收到 VSync 信号后，会通过 IPC 通知到 App 内。App 的 Runloop 在启动后会注册对应的 CFRunLoopSource 通过 mach_port 接收传过来的时钟信号通知，随后 Source 的回调会驱动整个 App 的动画与显示。

Core Animation 在 RunLoop 中注册了一个 Observer，监听了 BeforeWaiting 和 Exit 事件。这个 Observer 的优先级是 2000000，低于常见的其他 Observer。当一个触摸事件到来时，RunLoop 被唤醒，App 中的代码会执行一些操作，比如创建和调整视图层级、设置 UIView 的 frame、修改 CALayer 的透明度、为视图添加一个动画；这些操作最终都会被 CALayer 捕获，并通过 CATransaction 提交到一个中间状态去（CATransaction 的文档略有提到这些内容，但并不完整）。当上面所有操作结束后，RunLoop 即将进入休眠（或者退出）时，关注该事件的 Observer 都会得到通知。这时 CA 注册的那个 Observer 就会在回调中，把所有的中间状态合并提交到 GPU 去显示；如果此处有动画，CA 会通过 DisplayLink 等机制多次触发相关流程。

ASDK 在此处模拟了 Core Animation 的这个机制：所有针对 ASNode 的修改和提交，总有些任务是必需放入主线程执行的。当出现这种任务时，ASNode 会把任务用 ASAsyncTransaction(Group) 封装并提交到一个全局的容器去。ASDK 也在 RunLoop 中注册了一个 Observer，监视的事件和 CA 一样，但优先级比 CA 要低。当 RunLoop 进入休眠前、CA 处理完事件后，ASDK 就会执行该 loop 内提交的所有任务。具体代码见这个文件：ASAsyncTransactionGroup。

以上是前两周看的一些原理性的东西，代码太多太复杂还没研究懂~，总之ASDK就是能实现把异步、并发的操作同步到主线程去，并且获得不错的性能。只是这个库是有点重，使用需谨慎，要用就得用一套node的东西。如果没有特别多图片处理或者富文本编辑的话也体现不出它的优势。