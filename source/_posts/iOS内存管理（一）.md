---
title: iOS内存管理（一）
date: 2017-06-15 16:43:18
tags:
---
# 前言

这两篇博文是阅读《iOS与OS X多线程和内存管理》之后整理得来的，有错误之处，请指正！

# 引用计数

顾名思义，引用计数是指内存管理中对引用采用计数的技术。 对于引用计数，我们很自然的联想到“某处有某物多少多少”而将注意力放在计数上。但更客观、正确的思考方式是：
1. 自己生成的对象，自己持有

2. 非自己生成的对象，自己也能持有

3. 不再需要自己持有的对象时自己释放

4. 非自己持有的对象无法释放

上面出现了”生成“、”持有“、”释放“三个词，而在Object-C内存管理中，还应该加上”废弃“一词。各个词表示的Object-C方法表如下：

| right aligned | Objective-C方法                  |
| :------------ | ------------------------------ |
| 生成并持有对象       | alloc/new/copy/mutableCopy 等方法 |
| 持有对象          | retain 方法                      |
| 释放对象          | release 方法                     |
| 废弃对象          | dealloc 方法                     |

这些有关Objective-C内存管理的方法，并没包含在语言中，而是在Cocoa框架中。准确的说由Foundation框架类库的NSObject类负责内存管理职责。以下将对表格进行简单说明

# 自己生成的对象，自己持有

使用以下名称开头的方法名意味着自己生成的对象只有自己持有：

* alloc

* new

* copy

* mutableCopy


只要符合上述**以什么开头**的规则，且方法名满足**驼峰命名**要求，那么它们生成的对象也会被自己持有，比如：allocMyObject，newThatObject， copyThis，mutableCopyThat。相反，如果不符合驼峰命名，比如：allocate，newer，copying等方法所产生的对象则不会自己持有。

# 非自己生成的对象，自己也能持有

默认情况下，使用alloc/new/copy/mutableCopy以外的方法取得的对象，因为非自己生成，所以自己不是该对象的持有者。使用retain方法可以持有该对象：

```objective-c
//取得非（自己生成并持有）的对象，取得了对象的存在，但并不持有该对象
id obj = [NSMutableArray array];
//自己持有该对象
[obj retain];
```

通过retain方法，非自己生成的对象跟用alloc/new/copy/mutableCopy方法生成的对象一样，成为自己持有的了。

# 不在需要自己持有的对象时释放

自己持有的对象，当不再需要时，必须自己进行释放。释放使用release方法。

```objective-c
//自己生成并持有对象
id obj = [[NSObject alloc] init];
//释放对象
[obj release];
//此时，指向对象的指针仍然保留在变量中，但对象一经释放，就绝对不能再访问
```

如此，由自己生成并持有的对象就释放了。对于非自己生成并持有的对象，如果使用retain方法变为自己持有，那么自己也得负责释放。

```objective-c
//取得非（自己生成并持有）的对象，取得了对象的存在，但并不持有该对象
id obj = [NSMutableArray array];
//自己持有该对象
[obj retain];
//释放对象
[obj release];
```

思考一下，NSMutableArray类的array方法，因为它不是以alloc/new/copy/mutableCopy等名称开头，所以它不会持有生成的对象，但是它内部肯定调用了alloc来生成对象，从而取得了对象的存在。这是怎么实现的呢？？

```objective-c
+ (id)array
{
    //生成并自己持有对象
    id obj = [[NSMutableArray alloc] init];
    //取得对象的存在，但不再持有对象
    [obj autorelease];
    //如果没有调用autorelease方法，而是直接返回，那么obj没有调用release方法，所以没有被释放，外部调用方
    //因为方法名为object，所以调用方并不会持有返回对象，从而也不会释放它，因此就会造成内存泄露
    return obj;
}

//取得对象存在，但不持有该对象
id obj = [obj0 object];
```

通过**autorelease**方法，我们可以取得对象的存在，但却不持有该对象。同时，autorelease方法使得对象在超出指定的生存范围时能够自动并正确的释放。
当然，也能够通过retain方法将调用autorelease方法取得的对象变为自己持有。

# 无法释放非自己持有的对象

对于alloc/new/copy/mutableCopy方法生成并持有的对象，或是用retain方法持有的对象，持有者在不需要该对象的时候，有义务对其进行释放。而由此以外得到的对象，绝对不能释放。如果释放了非自己持有的对象，会Crash，Crash，Crash！！

```objective-c
id obj1 = [[NSObject alloc] init];
//自己持有的对象，自己进行释放
[obj1 release];

id obj2 = [obj0 object];
//非自己持有的对象，进行释放，后果Crash...
[obj2 release];
```

以上四项内容就是引用计数的思考方式，接下来会介绍在ARC下，这些规则会有哪些变化。

# 参考

1. 《Objective-C高级编程——iOS与OS X多线程和内存管理》