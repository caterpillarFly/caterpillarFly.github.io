---
title: ViewController那些事儿（二）
date: 2018-04-13 17:04:38
tags:
---

# view的状态

[上篇][1]文章提到了ViewController两个指定初始化方法

```objective-c
- (instancetype)initWithNibName:(nullable NSString *)nibNameOrNil bundle:(nullable NSBundle *)nibBundleOrNil;
- (nullable instancetype)initWithCoder:(NSCoder *)aDecoder;
```

那当VC分别用这两个方法完成初始化后，View是什么状态呢？已初始化还是未初始化？这得分情况：

当VC是通过Xib文件初始化时，

```objective-c
TestXibViewController *vc = (TestXibViewController *)[[[NSBundle mainBundle] loadNibNamed:@"TestXibViewController" owner:nil options:nil] firstObject];
//[self.navigationController pushViewController:vc animated:YES];
```

**loadNibNamed:owner:options:**方法结束后，View就已经被初始化好，且所有的**outlets/actions**已经设置好。

上篇文章提到，从UIStoryboard初始化的VC，关联过程和这不太一样。那从UIStoryboard初始化的VC，View处于什么状态呢？

```objective-c
UIStoryboard *storybaord = [UIStoryboard storyboardWithName:@"StoryboardVC" bundle:nil];
StoryboardVC *vc = [storybaord instantiateInitialViewController];
//[self.navigationController pushViewController:vc animated:YES];
```

UIStoryboard和UINib的处理方式不一样，虽然都是调用的**initWithCoder:**方法，但是结果却大不一样。

当调用**instantiateInitialViewController**方法得到VC时，虽然底层也调用了**instantiateWithOwner:options**方法，但是View并没有初始化，所有的**outlets/actions**也未关联。那他们是什么时候关联的呢？

当通过**initWithNibName:bundle:**方法初始化VC时，Vew也没有初始化，还是处于nil状态。为什么呢，参数里不是有nib文件名称吗？答案是苹果对内存的优化，所以使用延迟加载，**直到开发人员调用self.view**的时候，才会去初始化view。Storyboard也是类似的过程。（事实上，开发人员一般不会初始化完成就调用self.view，通常都是用导航控制器来执行push操作）

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

<img src="https://github.com/caterpillarFly/blogImages/blob/master/MGAlertViewControllerCrash.png?raw=true"/>

这个崩溃只出现在**iOS8**的系统上。工程里面，存在两个类，**MGAlertViewController**，**MGAlertView**。这两个类都有各自的xib文件，即：MGAlertViewController.xib和MGAlertView.xib。

代码中对MGAlertViewController的初始化，并没有传递nib名称，直接使用的：

```objective-c
[[MGAlertViewController alloc] init]
```

如果按照上述的xib查找规则，那么会找到MGAlertViewController.xib文件，这没有任何问题。

但是在iOS8的机型上，却找到了MGAlertView.xib文件，当对MGAlertView.xib文件中对象解固完成并进行关联时，没有设置view关联，所以发生了崩溃。

为什么iOS8和iOS9找到的xib文件不一样呢？只能猜测iOS8和iOS8以上系统的xib查找规则不同。没有找到英文资料说明这一差异，有篇[中文博客][1]，简单讲述了xib的查找规则。

总结下查找规则：

以MGAlertViewController为例，如果初始化时没传xib文件名称，那么系统首先会找与控制器名字一样，但是去掉Controller后缀，也就是MGAlertView.xib文件，找到了则使用这个xib文件。如果没找到，就会找与控制器名字一样的xib文件，即MGAlertViewController.xib。

以上就是View的初始化过程