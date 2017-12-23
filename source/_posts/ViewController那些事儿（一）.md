---
title: ViewController那些事儿（一）
date: 2017-12-10 10:20:56
tags:
---
# 简介
UIViewController是iOS开发中最常用的对象，负责管理视图层次。VC对象有一个属性：view，即我们所见的界面。那这个view是何时初始化的呢？VC初始化之后，view是啥状态呢？viewDidLoad是何时调用的？为啥从xib中拖出来的变量，都是weak的？这篇文章将会回答以上问题。
# VC指定初始化方法
所有的初始化方法都会调用它进行初始化操作，就是指定初始化方法。这里不做过多赘述。在Xcode里，指定初始化方法后会跟上**NS\_DESIGNATED\_INITIALIZER**标识符。那VC的指定初始化方法有哪些？

```objective-c
- (instancetype)initWithNibName:(nullable NSString *)nibNameOrNil bundle:(nullable NSBundle *)nibBundleOrNil;
- (nullable instancetype)initWithCoder:(NSCoder *)aDecoder;
```

这两个初始化方法有啥区别？通常情况下，VC可以通过以下两种方式初始化：

1. init

```objective-c
TestVC *vc = [[TestVC alloc] init];
XibTestVC *vc = [[XibTestVC alloc] initWithNibName:@"XibTestVC" bundle:nil];
```

2. nib文件

```objective-c
TestXibViewController *vc = (TestXibViewController *)[[[NSBundle mainBundle] loadNibNamed:@"TestXibViewController" owner:nil options:nil] firstObject];
UIStoryboard *storybaord = [UIStoryboard storyboardWithName:@"StoryboardVC" bundle:nil];
StoryboardVC *vc = [storybaord instantiateInitialViewController];
```

通过对指定初始化方法的讲解，我们知道，简单的init初始化，最终还是会调用**initWithNibName:bundle:**，当通过代码手动创建VC时，就会调用这个方法。

**initWithCoder:**方法，则是所有固化（archived）对象的初始化器。当需要从nib文件中加载对象时，就会调用该方法。当这个方法调用时，nib中的固化对象都会被解固，但是此时还并未和**outlets/actions**关联。

```
TestXibViewController *vc = (TestXibViewController *)[[[NSBundle mainBundle] loadNibNamed:@"TestXibViewController" owner:nil options:nil] firstObject];
XibTestVC *vc = [[XibTestVC alloc] initWithNibName:@"XibTestVC" bundle:nil];
```

上面的代码片段，可以发现我们创建了两个xib文件，XibTestVC.xib和TestXibViewController.xib。同样是xib文件，最终却调用完全不一样的初始化方法，为什么？

<img src="https://github.com/caterpillarFly/blogImages/blob/master/XibTestVC.xib.png?raw=true" width="60%" height="60%"/>

<img src="https://github.com/caterpillarFly/blogImages/blob/master/TestXibViewController.xib.png?raw=true" width="60%" height="60%"/>

创建VC时，勾选"Also create XIB file"，就会创建上图一样的xib。通过图片对比可以发现，上图只有View视图在xib中，而下面那张图，整个VC都在xib中。且两张图的file's owner，上图为指定的VC，即XibTestVC，下图为空。以上这些不同导致了VC在初始化时的不同。

上面说了，当initWithCoder:方法调用结束时，虽然nib中的对象都被解固了，但是还没有和VC中的outlets/actions等关联。那是何时进行关联的呢？

这就要提到两个方法：**instantiateWithOwner:options:** 和**awakeFromNib**。

instantiateWithOwner:options是UINib的类方法，它会将解固的对象和outlets/actions等进行关联。

如果说initWithCoder:方法是nib文件解固过程的开始，那么awakeFromNib就是这个过程的结尾。当nib文件中所有对象完成解固，并且和outlets/actions完成关联，系统就会调用awakeFromNib方法。

所以通过nib文件初始化VC的整个流程大致就是：

**NSBundle loadNibNamed:owner:options** -> **UIClassSwapper initWithCoder:** -> **VC initWithCoder** -> **UINib instantiateWithOwner:options:** -> **VC awakeFromNib**。

# view的状态

当VC初始化之后，其view处于什么状态呢？

通过上面的讲述，我们知道当VC通过initWithCoder:方法初始化时，调用完super解固后，view就已经初始化好了。当调用awakeFromNib后，整个解固过程就已经结束，所有的outlets/actions等都已经初始化完成。

但是通过initWithNibName:bundle:方法初始化VC时，并没有初始化VC对应的view属性，view还是处于nil。为什么呢？参数里不是有nib文件名称，bundle名称吗，答案是苹果对内存的优化，所以使用延迟加载，直到开发人员调用self.view的时候，才会去初始化view。

那view是如何初始化的呢？答案是loadView，该方法是VC用来初始化或者说加载view。具体加载过程如下：

```objective-c
- (void)loadView
{
  if (nib){ 
	loadNib;
  }
  else{
	createEmptyView;
  }
}
```

VC初始化时，如果传递了相应的nib参数，就加载指定的nib文件。如果没传，就按照一定的nib文件查找规则进行查找，找到了，那就加载找到的nib文件；没找到，就创建一个空的view。（[[UIVew alloc] init]）

## xib查找规则

那这个nib文件查找规则是怎样的呢？大多数教程说的都是直接查找与VC同名的nib文件。真是这样的吗？请看下面fabric统计的一个崩溃。

![](https://github.com/caterpillarFly/blogImages/blob/master/MGAlertViewControllerCrash.png?raw=true)

这个崩溃只出现在iOS8的系统上。工程里面，存在两个类，MGAlertViewController，MGAlertView。这两个类都有各自的xib文件，即：MGAlertViewController.xib和MGAlertView.xib。

代码中对MGAlertViewController的初始化，并没有传递nib名称，直接使用的[[MGAlertViewController alloc] init]。

如果按照上述的xib查找规则，那么会找到MGAlertViewController.xib文件，这没有任何问题。

但是在iOS8的机型上，却找到了MGAlertView.xib文件，当对MGAlertView.xib文件中对象解固完成并进行关联时，没有设置view，所以发生了崩溃。

为什么iOS8和iOS9找到的xib文件不一样呢？只能猜测iOS8和iOS8以上系统的xib查找规则不同。没有找到英文资料说明这一差异，有几篇[中文博客](http://www.cnblogs.com/Mike-zh/p/4430616.html)，简单讲述了xib的查找规则。

以MGAlertViewController为例，如果初始化时没传xib文件名称，那么系统首先会找与控制器名字一样，但是去掉Controller，也就是MGAlertView.xib文件，找到了则使用这个xib文件。如果没找到，就会找与控制器名字一样的xib文件，即MGAlertViewController.xib。

以上就是view的初始化过程。

# viewDidLoad何时调用

那VC初始化完成后，viewDidLoad方法何时调用呢？通常我们是点击某个按钮，创建一个VC，然后push。

```objective-c
- (IBAction)buttonClicked:(id)sender
{
    MyViewController *vc = [[MyViewController alloc] init];
	/** 设置vc需要的参数*/
	[self.navigationController pushViewController:vc animated:YES];
}
```

然后系统就会自动调用viewDidLoad方法了。

如果是通过nib文件初始化的VC，那么当awakeFromNib方法调用后，系统会自动调用viewDidLoad。但是通过init方法初始化的呢？

今天12月23号，星期六。上午终于开始学习科目二。

下午来公司，完成已经拖了很久的任务，即如何在多台电脑间同步hexo+git搭建的博客。