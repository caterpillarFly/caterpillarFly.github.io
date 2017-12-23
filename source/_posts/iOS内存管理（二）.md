---
title: iOS内存管理（二）
date: 2017-07-08 11:52:38
tags:
---

# 概要

ARC，即Automatic Reference Counting，字面意思“自动引用计数”， ARC只是自动的帮助我们处理“引用计数”相关的部分。关于引用计数的思考方式并没有发生变化，只是在源码的表述上稍有不同，发生了什么变化呢？首先要理解ARC中追加的所有权声明。

# 所有权修饰符

OC中，为了处理对象，将变量定义为id类型或各种对象类型，比如id，NSObject \*等。在ARC有效时，OC对象类型同C语言其它类型不同，其类型上必须附加所有权修饰符。所有权修饰符一共有四种，是：\_\_strong，\_\_weak，\_\_unsafe\_unretained，\_\_autoreleasing。附有这些修饰符的变量，会自动初始化为nil。以下对每种修饰符进行简单讲解。

## \_\_strong修饰符

\_\_strong修饰符表示对对象的“强引用”，持有强引用的变量在超出其作用域范围时被废弃，随着强引用的失效，其引用的对象会随之释放。（自动释放）
\_\_strong修饰符是OC所有对象默认的所有权修饰符。也就是说，以下代码是等效的：
```objective-c
{
  //自己生成并持有对象
  id obj = [[NSObject alloc] init];
  //和以下代码等价的
  id __strong obj = [[NSObject alloc] init];
}
//obj超出其作用域，强引用失效，所以自动释放obj所持有的对象。该对象的所有者不存在，因此废弃该对象
```

在ARC无效的情况下，代码如下：
```objective-c
/* ARC无效 */
{
    //自己生成并持有对象
    id obj = [[NSObject alloc] init];
    //不再需要对象时，释放
    [obj release];
}
```
上述代码取得的是自己生成并持有的对象，那取得非自己生成并持有的对象该怎么办呢？
```objective-c
{
  //自己非生成并持有对象
  id __strong obj = [NSMutableArray array];
  //因为变量obj为强引用，所以它自己持有对象
}
//obj超出其作用域，强引用失效，所以自动释放obj所持有的对象。该对象的所有者不存在，因此废弃该对象
```
其实上述代码，最终会被编译器转换成：

```objective-c
id obj = objc_msgSend(NSMutableArray, @selector(array));
objc_retainAutoreleasedReturnValue(obj);
objc_release(obj);
```

可以看出，\_\_strong修饰符可以自动持有对象，尽管对象非自己生成并持有的。
通过上述几段代码，可以发现，“自己生成的对象，自己持有”和“非自己生成的对象，自己也能持有”这两条只需通过对带\_\_strong修饰符的变量赋值即可完成。而“不再需要持有的对象时自己释放”，当变量超出作用域范围（或者对变量进行重新赋值）时，会自动触发。最后一条“非自己持有的对象，自己不能释放“，由于不能调用release方法，所以这条自然也满足。

## \_\_weak修饰符
看起来好像通过\_\_strong修饰符就能很好的管理内存了，但是仅有\_\_strong是不够的，它无法解决引用计数式内存管理中无法避免的”**循环引用**“问题，所以才有\_\_weak修饰符。
\_\_weak与\_\_strong相反，提供弱引用。弱引用不能持有对象实例。\_\_weak修饰符还有一个优点，即在持有某对象的弱引用时，若该对象被废弃，则次弱引用将自动置为nil，反过来，也可以通过检查有\_\_weak修饰符的变量是否为nil，来判断被赋值的对象是否被废弃。
使用\_\_weak修饰符时需要注意，不能直接生成对象并对其赋值，如下：
```objective-c
id __weak obj = [[NSObject alloc] init];
```
[[NSObject alloc] init]属于自己生成并持有的对象，但\_\_weak修饰符不能持有对象实例，所以生成的对象会被立即释放，编译器对此会发出警告。
```objective-c
{
  //obj0为强引用，自己生成并持有对象
  id __strong obj0 = [[NSObject alloc] init];
  //obj1持有生成对象的弱引用
  id __weak obj1 = obj0;
}
//obj0超出作用域，强引用失效，所以自动释放它持有的对象。因为对象没有持有者了，所以废弃该对象
```
那又是如何做到\_\_weak变量，在它弱引用的对象释放后，自动置为nil的呢？

答案是系统会维持一张weak表，也是散列表，该对象作为建，弱引用它的weak变量为值，如果有多个弱引用的话，则会形成类似链表的结构，所以当该对象自动释放时，会遍历这个weak表，找出它对应的弱引用，然后将它们置为nil，并删除相应的键值记录。由此可知，如果大量使用附有\_\_weak修饰符的变量，会消耗相应的CPU资源。

## \_\_unsafe\_unretained修饰符

正如其名unsafe所言，是不安全的所有权修饰符。尽管ARC式的内存管理是编译器的工作，但附有\_\_unsafe\_unretained修饰符的变量不属于编译器的内存管理对象。它的作用和\_\_weak修饰符变量类似，但它并不持有对象的强引用或弱引用，而且它表示的对象一旦被废弃，它并不会被置为nil，而是变成野指针，所以尽量不要使用\_\_unsafe\_unretained。

## \_\_autoreleasing修饰符

无论ARC是否有效，都可使用非公开函数**\_objc\_autoreleasingPoolPrint()**来打印Pool信息。

先看一下，在非ARC的情况下，怎么使用autorelease的。

```objective-c
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
id obj = [[NSObject alloc] init];
[obj autorelease];
[pool drain];
```
NSAutoreleasePool对象的生命周期，相当于变量作用域。对该周期内，所有调用过autorelease方法的对象，在pool被废弃时，会自动向它发送release消息。
在ARC情况下，代码可改写如下：
```objective-c
@autoreleasepool {
    id __autoreleasing obj = [[NSObject alloc] init];
}
```
@autoreleasepool{}来代替NSAutoreleasePool类对象的生成、持有以及废弃这一范围。通过给变量附加\_\_autoreleasing修饰符，来代替向变量发送autorelease消息，即对象被注册到autoreleasepool中。
上一篇博文说道，如何获取非自己生成并持有的对象，主要就是通过在非alloc、new、copy、mutableCopy等方法返回的对象上调用autorelease方法。那在ARC的情况下又是怎么样的呢？

```objective-c
+ (id)array
{
    return [[NSMutableArray alloc] init];
}
```

但是这段代码，经过编译器的转换之后，将变成：

```
+ (id)array
{
    id obj = objc_msgSend(NSMutableArray, @selector(alloc));
    objc_msgSend(obj, @selector(init));
    return objc_autoreleaseReturnValue(obj);
}
```

转换的最后，调用了一个函数：**objc\_autoreleaseReturnValue**，它与一般的**objc\_autorelease**函数不同，它不仅限于注册对象到autoreleasePool中。它还有一个与之配对的函数：**objc\_retainAutoreleasedReturnValue**。这对函数主要用于极致优化程序运行，怎么个优化法呢？

**objc\_autoreleaseReturnValue**函数会检查使用该函数的方法或函数调用方的执行命令列表，如果调用方在调用了该函数后紧接着调用**objc\_retainAutoreleasedReturnValue**函数，那么久不讲返回的对象注册到autoreleasePool中，而是直接传递到函数的调用方。通过这两个函数，就可以不将对象注册到autoreleasePool中而直接传递，这一过程达到了最优化。

所以，array方法返回的对象，不一定注册到了autoreleasePool中，那么看以下代码：

```objective-c
@autoreleasePool{
    id __strong obj = [NSMutableArray array];
    _objc_autoreleasingPoolPrint();     //打印当前释放池的信息
}
```

猜想一下，Pool中是否有obj这一对象呢？经过试验，答案是没有，以上的优化过程可以解释为什么。

前面说到，id obj其实就是id \_\_strong obj，那么id \*obj呢，也是id \_\_strong \*obj吗？答案是NO，它对应的是：id \_\_autoreleasing \*obj；像这样，id的指针或对象的指针，在没有指定修饰符的情况下，默认会加上\_\_autoreleasing修饰符。

我们经常看到一些方法，在参数里面返回error信息，方法声明如下：

```objective-c
- (BOOL)performOperationWithError:(NSError **)error;
//等价于
- (BOOL)performOperationWithError:(NSError * __autoreleasing *)error;
```

调用如下：

```objective-c
NSError *error = nil;   //等价于NSError __strong *error
BOOL result = [obj performOperationWithError:&error];
```

苹果自身很多API也是通过这样的方式返回错误信息的。

但是，如果直接赋值，那么会产生编译器错误，代码如下：

```objective-c
NSError *error = nil;
NSError **pError = &error;      //编译器会报错
```

这是因为，赋值给**指针对象**时，对象的所有权修饰符必须保持一致。因为error变量附加有\_\_strong修饰符，而pError则附加有\_\_autoreleasing修饰符，所以报错。可更改为：NSError  \* \_\_strong  \*pError = &error即可。

但是，之前的函数调用，使用\_\_strong修饰符的error变量，并没有进行转换，也没有报错呀，这是为什么呢？这是因为编译器自动进行了如下转化：

```objective-c
NSError __strong *error = nil;
NSError __autoreleasing *tmpError = error;
BOOL result = [obj performOperationWithError:&error];
```

# 引用计数

在ARC的情况下，怎么知道对象的引用计数值呢？

**uintptr_t  _objc_rootRetainCount(id obj);**

该函数可以取得对象的计数值。但是，该函数是非线程安全的，且对于已释放的对象和不正确的对象类型，返回的值也是不准确的。所以该函数只提供一个参考值。

# 疑惑

1. 书籍上说，在访问附有\_\_weak修饰符的变量时，实际上必定要访问注册到autoreleasePool中的对象，比如：

   ```objective-c
   id obj0 = [[NSObject alloc] init];
   id __weak obj1 = obj0;
   NSLog(@"class=%@",[obj1 class]);
   ```

   上述代码和一下代码相同：

   ```objective-c
   id __weak obj1 = obj0;
   id __autoreleasing tmp = obj1;
   NSLog(@"class=%@",[tmp class]);
   ```

   但实际试验结果和书上所说有所出入，如果把两段代码放入@autoreleasePool{}里面，然后打印Pool的信      息，就会发现下面的代码，池子里面包含了对象obj0，但是上面的代码则没有包含。所以访问\_\_weak变量，实际上必定访问注册到autoreleasePool中的对象，对此持保留态度。

2. 我们一般都是这样使用：

   ```objective-c
   @autoreleasepool {
       id obj = [[NSObject alloc] init];
        id mArray = [NSMutableArray array];
   }
   ```

   经过上面讲述的优化过程，知道obj和mArray并不会注册到autoreleasePool中，打印Pool信息，也确实没发现这两个对象。既然没注册到Pool里面，那么外面的@autoreleasepool{}块还有用吗？

