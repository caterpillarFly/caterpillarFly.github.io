---
title: ViewController那些事儿（一）
date: 2017-12-10 10:20:56
tags:
---
# 简介
UIViewController是iOS开发中最常用的对象，负责管理视图层次，即视图容器
VC对象有一个属性：view，即我们所见的界面。

故VC只是一个容器，它并不展示任何内容，展示内容都是View负责的。

那这个view是何时初始化的呢？VC初始化之后，view是啥状态呢？viewDidLoad是何时调用的？为啥从xib中拖出来的变量，都是weak的？这篇文章将会回答以上问题。

# VC指定初始化方法
在Xcode里，指定初始化方法后会跟上**NS\_DESIGNATED\_INITIALIZER**标识符。那VC的指定初始化方法有哪些？
```objective-c
- (instancetype)initWithNibName:(nullable NSString *)nibNameOrNil bundle:(nullable NSBundle *)nibBundleOrNil;
- (nullable instancetype)initWithCoder:(NSCoder *)aDecoder;
```
这两个初始化方法有啥区别？

通常情况下，VC可以通过以下两种方式初始化：

1. init
```objective-c
TestVC *vc = [[TestVC alloc] init];
XibTestVC *vc = [[XibTestVC alloc] initWithNibName:@"XibTestVC" bundle:nil];
```
当通过代码手动创建VC时，就会调用调用**initWithNibName:bundle:**这个方法。

2. nib文件

```objective-c
TestXibViewController *vc = (TestXibViewController *)[[[NSBundle mainBundle] loadNibNamed:@"TestXibViewController" owner:nil options:nil] firstObject];
UIStoryboard *storybaord = [UIStoryboard storyboardWithName:@"StoryboardVC" bundle:nil];
StoryboardVC *vc = [storybaord instantiateInitialViewController];
```

**initWithCoder:**方法，则是所有固化（archived）对象的初始化器。当需要从nib文件中加载对象时，就会调用该方法。当这个方法调用时，nib中的固化对象都会被解固，但是此时还并未和**outlets/actions**关联。

那是何时进行关联的呢？

这就要提到两个方法：**instantiateWithOwner:options:** 和**awakeFromNib**。

**instantiateWithOwner:options**是UINib的类方法，它会将解固的对象和outlets/actions等进行关联。

当nib文件中所有对象完成解固，并且和outlets/actions完成关联，系统就会调用awakeFromNib方法。

所以通过nib文件初始化VC的整个流程大致就是：

**NSBundle loadNibNamed:owner:options** -\> **UIClassSwapper initWithCoder:** -\> **VC initWithCoder** -\> **UINib instantiateWithOwner:options:** -\> **VC awakeFromNib**。

从UIStoryboard初始化的VC，关联过程和这不太一样，具体关联过程下章讲到View状态时再解释。

另外，上面的两段初始化代码，可以看出，我们创建了两个xib文件，XibTestVC.xib和TestXibViewController.xib。同样是xib文件，为啥最后采用完全不同的初始化方法呢？

<img src="https://github.com/caterpillarFly/blogImages/blob/master/XibTestVC.xib.png?raw=true" width="60%" height="60%"/>

<img src="https://github.com/caterpillarFly/blogImages/blob/master/TestXibViewController.xib.png?raw=true" width="60%" height="60%"/>

通过图片对比可以发现，上图只有View视图在xib中，而下图，整个VC（包括它的视图）都在xib中。

当初始化XibTestVC时，并不需要初始化它的视图，所以只是传入了一个Xib文件名，这是后续初始化View需要的

```objective-c
XibTestVC *vc = [[XibTestVC alloc] initWithNibName:@"XibTestVC" bundle:nil];
```

但是TestXibViewController初始化时，因为它的VC和视图都在Xib文件中，所以只能通过Xib文件来初始化VC和视图。

```objective-c
TestXibViewController *vc = (TestXibViewController *)[[[NSBundle mainBundle] loadNibNamed:@"TestXibViewController" owner:nil options:nil] firstObject];
```

[1]:	http://www.cnblogs.com/Mike-zh/p/4430616.html

[image-1]:	https://github.com/caterpillarFly/blogImages/blob/master/MGAlertViewControllerCrash.png?raw=true