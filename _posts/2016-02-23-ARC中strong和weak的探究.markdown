---
title:  "ARC中strong和weak的探究"
date:   2016-02-23 17:11:02
categories: [iOS, 内存管理]
tags: [iOS, 内存管理]
---

曾几何时，自己也是对 strong/retain/weak等晕头转向，今天看到了自己之前整理的关于ARC中的strong指针和weak指针的 demo和几篇文章，所以便来总结一下。

[![Contact](https://img.shields.io/badge/contact-wangyanchang21-green.svg)](https://github.com/wangyanchang21)

------

- [简介](#简介)
- [ARC原理](#arc原理)
- [strong指针和weak指针区别](#strong指针和weak指针区别)
- [代码实践](#代码实践)
	- [第一次测试](#第一次测试)
	- [第二次测试](#第二次测试)
	- [第三次测试(拓展)](#第三次测试拓展)
- [相关资料](#相关资料)

------


## 简介

`ARC`是自iOS5之后增加的新特性，完全消除了手动管理内存的烦琐，编译器会自动在适当的地方插入适当的`retain`、`release`、`autorelease`语句。你不再需要担心内存管理，因为编译器为你处理了一切。
注意：`ARC` 是编译器特性，而不是 iOS 运行时特性(除了weak指针系统)，它也不是类似于其它语言中的垃圾收集器。因此`ARC`和手动内存管理性能是一样的，有时还能更加快速，因为编译器还可以执行某些优化。

## ARC原理
在引用计数架构下, 对象有个计数器，统计着对象被多少个强引用指针变量持有，即引用计数。当没有强引用指针指向该对象时，即引用计数降为0时，该对象会自动释放。

## strong指针和weak指针区别

`strong指针`可以理解为对象的拥有者，`weak指针`可以指向某个对象，但不属于对象的拥有者。

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/strongweak/20160223142751243.png" width="70%" img/>
</center>

如上图面的关系图，总结为以下几点:
1.`strong指针`和`weak指针`都指向同一个对象，但`weak指针`不是拥有者。
2.如果`strong指针`不在指向该对象，则该对象就没有拥有者，就会被释放，此时 `weak指针`会自动变成`nil`，称为空指针。 所以，weak型的指针变量自动变为nil是非常方便的，这样避免了`weak指针`继续指向已释放对象，避免了野指针相关问题的产生。
3.对于解决循环引用的问题来说，`weak指针`是极其便利的。在`delegate模式`下，一个weak就能解决循环引用的问题，且不会造成野指针的后顾之忧。

## 代码实践

为了使理解更清晰，整理了一个小的 demo，下面再来看一下。 先简单说下思路，`ARC` 自动管理对象的释放，所以就涉及到一个自动释放池的概念，这里就不赘述了。 在自动释放池内，创建三组或强或弱的对比属性，创建时赋值，打印。 然后在自动释放池外，即通过屏幕按钮点击事件实现，再重新打印之前的所有属性，再次对比，验证以上说法。

### 第一次测试

```swift
#import "ViewController.h"

@interface ViewController ()
//第一组
@property (nonatomic, strong) NSString *groupOne_strong;
@property (nonatomic, weak) NSString *groupOne_weak;
//第二组
@property (nonatomic, strong) NSString *groupTwo_strong1;
@property (nonatomic, strong) NSString *groupTwo_strong2;
//第三组
@property (nonatomic, strong) NSString *groupThree_strong;
@property (nonatomic, weak) NSString *groupThree_weak;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    @autoreleasepool {
        //正常的强引/弱引用对比
        [self strongAndWeakCompare];
        //强引用赋值给强引用,将第一个强引用置为 nil
        [self strongGiveToStrong];
        //强引用赋值给弱引用,将强引用置为 nil
        [self strongGiveToWeak];
    }
}

//正常的强引/弱引用对比
- (void)strongAndWeakCompare
{
    self.groupOne_strong = [NSString stringWithFormat:@"强弱对比"];
    self.groupOne_weak = [NSString stringWithFormat:@"strongAndWeakCompare"];
    NSLog(@"第1组强弱对比:srong=%@-<%p>,weak=%@-<%p>",_groupOne_strong,_groupOne_strong,_groupOne_weak,_groupOne_weak);
}
//强引用赋值给强引用,将第一个强引用置为 nil
- (void)strongGiveToStrong
{
    self.groupTwo_strong1 = [NSString stringWithFormat:@"强赋强"];
    self.groupTwo_strong2 = _groupTwo_strong1;
    _groupTwo_strong1 = nil;
    NSLog(@"第2组强赋强:srong1=%@-<%p>,strong2=%@-<%p>",_groupTwo_strong1,_groupTwo_strong1,_groupTwo_strong2,_groupTwo_strong2);
}
//强引用赋值给弱引用,将强引用置为 nil
- (void)strongGiveToWeak
{
    self.groupThree_strong = [NSString stringWithFormat:@"强赋弱"];
    self.groupThree_weak = _groupThree_strong;
    _groupThree_strong = nil;
    NSLog(@"第3组强赋弱:srong=%@-<%p>,weak=%@-<%p>",_groupThree_strong,_groupThree_strong,_groupThree_weak,_groupThree_weak);
}
//待循环池中的对象销毁后,进行打印验证
- (IBAction)click:(id)sender
{
    NSLog(@"第1组强弱对比:srong=%@-<%p>,weak=%@-<%p>",_groupOne_strong,_groupOne_strong,_groupOne_weak,_groupOne_weak);
    NSLog(@"第2组强赋强:srong1=%@-<%p>,strong2=%@-<%p>",_groupTwo_strong1,_groupTwo_strong1,_groupTwo_strong2,_groupTwo_strong2);
    NSLog(@"第3组强赋弱:srong=%@-<%p>,weak=%@-<%p>",_groupThree_strong,_groupThree_strong,_groupThree_weak,_groupThree_weak);
}
```

PS:在自动释放池内还未释放时(这里也可以不写@autoreleasepool，因为在main函数文件中已然添加)。 在这里这个自动释放池只是表明了一个对象的释放时机，`MRC`中对象的释放时机更加灵活，但 weak不可以在`MRC`情况下使用。

#### 第一次打印结果

先看打印结果，自动释放池内打印结果:

```
第1组强弱对比:srong=强弱对比-<0x7fe70945d320>, weak=strongAndWeakCompare-<0x7fe709456960>
第2组强赋强:srong1=(null)-<0x0>,strong2=强赋强-<0x7fe70945c460>
第3组强赋弱:srong=(null)-<0x0>,weak=强赋弱-<0x7fe709459750>
```

点击屏幕按钮，自动释放池外打印结果:

```
第1组: 强弱对比:srong=强弱对比-<0x7fe70945d320>,weak=(null)-<0x0>
第2组: 强赋强:srong1=(null)-<0x0>,strong2=强赋强-<0x7fe70945c460>
第3组: 强赋弱:srong=(null)-<0x0>,weak=(null)-<0x0>
```

根据打印结果发现，正如一开始我们所说的一样。 但是在第3组中 `strong指针`在置为 nil 后立即打印`weak指针`，其不为 nil，这就是因为自动释放池中还没有销毁，所以此时不会为 nil。

### 第二次测试

但我在整理的时候，发现了一个问题，一个 strong和 weak 的对比，不会像刚刚这样，难道是种特殊情况吗，来看一下。

属性中添加第四组:

```swift
//第四组
@property (nonatomic, strong)NSString *groupFour_strong;
@property (nonatomic, weak)NSString *groupFour_weak;
```

自动释放池内执行以下代码:

```swift
 //强引用于常量区，将强引用置为 nil
- (void)constantStrongGiveToWeak
{
    self.groupFour_strong = @"常量区赋值";
    //    self.groupFour_strong = [NSString stringWithFormat:@"a"];
    self.groupFour_weak = _groupFour_strong;
    _groupFour_strong = nil;
    NSLog(@"第4组常量区赋值:srong=%@-<%p>,weak=%@-<%p>",_groupFour_strong,_groupFour_strong,_groupFour_weak,_groupFour_weak);
}
```

自动释放池外点击事件中添加代码:

```swift
//待循环池中的对象销毁后,进行打印验证
- (IBAction)click:(id)sender
{
    NSLog(@"第4组常量区赋值:srong=%@-<%p>,weak=%@-<%p>",_groupFour_strong,_groupFour_strong,_groupFour_weak,_groupFour_weak);
}
```

#### 第二次测试打印结果

来看一下打印结果 (自动释放池内/外):

```
第4组常量区赋值:srong=(null)-<0x0>,weak=常量区赋值-<0x107a55160>
第4组常量区赋值:srong=(null)-<0x0>,weak=常量区赋值-<0x107a55160>
```

这时，会发现，自动释放池外打印结果 `weak指针`竟然不为 nil。 因为第一次测试中字符串的都是栈区创建的，所以也是在栈区来测试的，若是直接赋值字符创，如 self.groupOne_strong = @"强弱对比"，这样开辟的空间在常量区，所以即使将其置为 nil，其也不会释放。 


### 第三次测试(拓展)

添加属性，声明特性为 weak 类型的 UIView

```swift
//声明关键字为 weak 的 View 作为成员变量
@property (nonatomic, weak) UIView *red_weak_view;
```

自动释放池内执行以下代码:

```swift
//创建 weak View
- (void)createWeakView
{
    UIView *temp_view = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 100, 100)];
    temp_view.backgroundColor = [UIColor redColor];
    [self.view addSubview:temp_view];
    self.red_weak_view = temp_view;
}
```

自动释放池外点击事件中添加代码:

```swift
//待循环池中的对象销毁后,进行打印验证
- (IBAction)click:(id)sender
{
    NSLog(@"redview=%@",_red_weak_view);
}
```

#### 第三次测试打印结果

自动释放池外打印结果:

```
redview=< UIView: 0x7fe63a4aeaf0; frame = (0 0; 100 100); layer = < CALayer: 0x7fe63a4a2650 > >
```

像上面这样直接创建的 view，系统自动定义为 strong类型的，只是其作用域小。 当其addSubview后，其引用计数+1，并不会释放，所以属性中的weak view可以全局使用。

我曾见过有人这样使用，但我并不明白其中的优势所在，若大神懂得，望不吝赐教。 所以我在开发中会直接声明一个 strong 的属性，如果忘记addSubview的话，其便不能得到。



## 相关资料

[ARC到底帮我们做了哪些工作?](https://dcsnail.blog.csdn.net/article/details/79461511)   
[高效 OC开发之内存管理](https://blog.csdn.net/wangyanchang21/article/details/79356164)   
 [ARC下的注意事项](http://blog.csdn.net/wangyanchang21/article/details/50731028)   
[ARC指南1 - strong和weak指针](http://blog.csdn.net/q199109106q/article/details/8565017)   


-------

欢迎指正, [wangyanchang21](https://github.com/wangyanchang21).


