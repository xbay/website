date: 2016-01-26 18:15:50
author: [raven.zhang]
title: ASDK基本原理
tags: [iOS]
---

AsyncDisplayKit是Facebook开源的一套用于提高iOS界面流畅的UI框架。
--------------------------------------------------------------

AsyncDisplayKit的基本单元是node，ASDisplayNode是对于UIKit的UIView以及CALayer的抽象。但是它不像UIView一样只能在主线程初始化，下面是AsyncDisplayKit和UIKit的一些类对应关系。

ASDisplayNode. Counterpart to UIView — subclass to make custom nodes.
ASControlNode. Analogous to UIControl — subclass to make buttons.
ASImageNode. Like UIImageView — decodes images asynchronously.
ASTextNode. Like UITextView — built on TextKit with full-featured rich text support.
ASTableView and ASCollectionView. UITableView and UICollectionView subclasses that support nodes.

ASDisplayNode 是线程安全的，它可以在后台线程创建和修改。Node 刚创建时，并不会在内部新建 UIView 和 CALayer，直到第一次在主线程访问view或 layer 属性时，它才会在内部生成对应的对象。当它的属性（比如frame/transform）改变后，它并不会立刻同步到其持有的 view 或 layer 去，而是把被改变的属性保存到内部的一个中间变量，稍后在需要时，再通过某个机制一次性设置到内部的 view 或 layer。

把布局、绘制、图像抽出主线程。
而且避免重复绘制任务，以及及时取消无效的任务（无效任务指的是上一次发起的绘制还没完成但是已经不需要了就及时取消）。