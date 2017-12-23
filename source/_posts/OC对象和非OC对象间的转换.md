---
title: OC对象和Core Foundation对象间的转换
date: 2017-07-18 19:47:05
tags:
---

# 简介

Core Foundation框架，是一组由C语言编写的接口，它所使用的对象，称之为Core Foundation对象，也是使用引用计数。CF对象与OC对象的区别很小，且可以混合使用。Foundation框架的API生成的对象，可以用Core Foundataion框架的API释放，反之亦然。

# 内存管理

Core Foundation框架提供了一下几个函数来进行内存管理，**CFBridgingRetain**，**CFBridgingRelease**。这两个函数分别和**\_\_bridge\_retain**，**\_\_bridge\_release**等价。另外，CF框架还提供了一个函数，来查看对象的引用计数，即：**CFGetRetainCount**。下面分别举例说明这几个函数的用法及内存管理情况：

## \_\_bridge

以下代码，将OC对象与void\*指针进行相互转换。

```objective-c
void *p;
{
  id obj = [[NSObject alloc] init];
  p = (__bridge void *)obj;         //id对象转换成void*指针
  id op = (__bridge id)p;           //void*指针转换成id对象
}
//这行代码会崩溃，因为指针指向的对象已经释放，即指针是野指针了
id obj = (__bridge id)p;
```

转换为void \*的\_\_bridge转换，其安全性与赋值给\_\_unsafe\_unretained修饰符相近，如果不注意赋值对象的所有者，就会因悬挂指针而导致程序崩溃。

## \_\_bridge\_retained

\_\_bridge\_retained主要是将OC对象转换为CF对象。它和CFBridgingRetain等价，\_\_bridge\_retained转换，可使要转换赋值的变量也持有所赋值的对象。下面通过代码说明对象内存的管理情况：

```objective-c
CFMutableArrayRef cfObject = NULL;
{
    id obj = [[NSMutableArray alloc] init];
    //cfobject也持有对象，引用+1
    cfobject = CFBridgingRetain(obj);       
    //该代码和下行转换代码等价
    //cfobject = (__bridge_retained CFTypeRef)obj;
    printf("retain count = %ld\n", CFGetRetainCount(cfobj));
}
printf("retain count after the scope = %ld\n", CFGetRetainCount(cfobj));
CFRelease(cfobject);
```

代码运行结果如下：

```
retain count = 2
retain count after the scope = 1
```

可以看出，通过\_\_bridge\_retained转换，对象的引用计数加1了，而且，如果不调用CFRelease进行释放的话，会造成内存泄露。可以看出，Foundation框架的API生成的OC对象，可以作为Core Foundation对象来使用，也可以通过CFRelease来释放。

那如果使用\_\_bridge代替\_\_bridge\_retained，会是什么情况呢？

```objective-c
CFMutableArrayRef cfObject = NULL;
{
    id obj = [[NSMutableArray alloc] init];
    //cfobject并不持有对象
    cfobject = (__bridge CFTypeRef)obj;
    printf("retain count = %ld\n", CFGetRetainCount(cfobj));
}
printf("retain count after the scope = %ld\n", CFGetRetainCount(cfobj));
CFRelease(cfobject);
```

结果输出：retain count = 1，并在第二个打印语句那崩溃。为什么呢？因为前面说过，\_\_bridge转换并不会改变对象的持有状况，所以引用计数还是1。且超出作用域之后，cfobject就变为野指针。访问野指针造成程序崩溃。

## \_\_bridge\_transfer

与\_\_bridge\_retained相反，\_\_bridge\_transfer将CF对象转换成OC对象。并且，被转换的变量，它所持有的对象，在它被赋值给转换目标后随之释放。看以下代码：

```objective-c
{
    CFMutableArrayRef cfobj = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
    printf("retain count = %ld\n", CFGetRetainCount(cfobj));
    id obj = CFBridgingRelease(cfobj);
    //这个转换和以下转换等价：
    //id obj = (__bridge_transfer id)cfobj;
    //注意，虽然cfobj不再持有对象的引用，但是指针还是有效的，还是其正常作用域范围
    printf("retain count after the cast = %ld\n", CFGetRetainCount(cfobj));    
}
```

输出结果如下：

```
retain count = 1
retain count after the cast = 1
```

经过\_\_bridge\_transfer转换，对象的引用计数没有改变，且CF框架创建的对象cfobj，并没有调用CFRelease进行释放，这是因为经过\_\_bridge\_transfer转换后，cfobj不再持有对象的引用，故而不需要进行释放。但obj持有对象引用，它的释放是有ARC自动进行的。

同样，如果用\_\_bridge来代替\_\_bridge\_transfer，将会是什么情况呢？

```objective-c
{
    CFMutableArrayRef cfobj = CFArrayCreateMutable(kCFAllocatorDefault, 0, NULL);
    printf("retain count = %ld\n", CFGetRetainCount(cfobj));
    id obj = (__bridge id)cfobj;
    printf("retain count after the cast = %ld\n", CFGetRetainCount(cfobj));    
}
```

输出结果如下：

```objective-c
retain count = 1
retain count after the cast = 2
```

为什么引用计数加1了？因为id obj = (\_\_bridge id)cfobj;其实和 id \_\_strong obj = (\_\_bridge id)cfobj等价。所以obj会持有对象的强引用，从而引用计数加1。到{}结束时，OC对象obj超出作用域，其强引用失效，对象引用计数减一。但是，cfobj对象还持有引用，它也没有调用CFRelease进行释放，对象的引用计数还为1，从而造成内存泄露。

所以，在将CF对象通过\_\_bridge转换成OC对象时，一定要注意对象的释放问题。