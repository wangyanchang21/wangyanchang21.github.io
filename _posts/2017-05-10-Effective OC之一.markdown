---
title:  "高效 OC开发之熟悉Objective-C"
date:   2017-05-10 23:23:30
categories: [iOS, Effective OC]
tags: [iOS, Effective OC]
---

关于头文件，语法糖字面量，常量和预编译的定义等的高效使用。

[![Contact](https://img.shields.io/badge/contact-wangyanchang21-green.svg)](https://github.com/wangyanchang21)  

------

- [① OC起源](#-oc起源)
- [② 在类的头文件中尽量少引入其他头文件](#-在类的头文件中尽量少引入其他头文件)
- [③ 多用字面量语法(语法糖)](#-多用字面量语法语法糖)
- [④ 多用常量类型,少用#define预处理指令](#-多用常量类型-少用define预处理指令)
- [⑤ 用枚举表示状态、选项、状态码](#-用枚举表示状态选项状态码)
- [相关资料](#相关资料)

-------


## ① OC起源

`Objective-C`是C语言添加了`面向对象`特性, 是其超集(superset)。
OC语言使用的是`消息机制`, 并不是通过函数调用方法, 而是通过发送消息。Objective-C 使用动态绑定的消息结构, 只有在运行时才会检查对象类型并处理, 其过程叫做`动态绑定`(dynamic binding)。接收一条消息之后, 究竟应执行何种代码, 由`运行时环境`而非编译器来决定。OC的重要工作都由`运行时组件`(runtime component)而非编译器来完成。

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-1/20170510173004777.png" width="70%" img/>
</center>

所有OC中的`对象内存`总是分配在`堆空间`中的, 而对象本身的`指针变量`内存分配在`栈区`中, 而且指针本身的内存在32位架构计算机上是4字节, 64位架构计算机上是8字节的。另外, 非对象类型的变量也存储于栈区中, 如基本数据类型(int, double, char...), 结构体(CGRect, CGRange...)等。


## ② 在类的头文件中尽量少引入其他头文件

OC中有`头文件`(header file)和`实现文件`(implementation file), 也就是所用的`.h`和`.m`文件。

将我们引入头文件的时机尽量延后, 只在确实有需要时才引入, 这样就可以减少类的使用者所需引入的头文件数量。 这样做的好处是, 减少编译时间的效果, 降低类之间的耦合性。

在一般的情况下, 可以配合`向前声明`(forward declaring)来达到这样的效果。用法如下:

``` swift
// .h
#import <UIKit/UIKit.h>

@class BankModel;

@interface BankLimitCell : UITableViewCell
@property (nonatomic, strong) BankModel *model;
@end
```

``` swift
// .m
#import "BankLimitCell.h"
#import "BankModel.h"

@interface BankLimitCell ()
@property (nonatomic, strong) UILabel *bankNameLable;
@end
```

在头文件中通过`@class`来声明这个类, 而不是直接导入头文件; 在实现文件中, 就必须要引入其头文件了。这样就延后了导入头文件的时机。

但是有时必须要在头文件中引入其他头文件。  
1.如果你写的类继承自某个父类, 则必须引入定义那个父类的头文件。  
2.如果要声明你写的类遵从某个协议(protocol), 那么该协议必须要有完整定义, 所以不能使用向前声明。  

``` swift
// .h  
#import "Animal.h"
#import "EatDelegate.h"

@interface Fish : Animal<EatDelegate>
@property (nonatomic, copy) NSString *name;
@end
```

针对于上面第2种情况, 其实也有别的方案, 可以将协议服从写在实现文件的类的延伸中去。

``` swift
#import "EatDelegate.h"

@interface Fish ()<EatDelegate>
@end

@implementation Fish
@end
```


## ③ 多用字面量语法(语法糖)

正常情况下, 在数组或者字典进行创建时, 遇到第一个为`nil`的时候, 会认为添加结束。但用字面量来创建数组或者字典的时候, 如果对象有为`nil`的话, 会造成crash的后果。所以根据这个微小的差别，在字面量语言中总能及时在调试中发现问题， 让我们去修改。

``` swift 
[NSArray arrayWithObjects:object1,object2,object3, nil];
@[object1,object2,object3];
```

具体的一些用法参考我之前的博客：[OC语法糖总结](http://blog.csdn.net/wangyanchang21/article/details/50674590)

 - 1.应该使用字面量语法来创建`字符串`、`数值`、`数组`、`字典`。与创建此类对象的常规方法相比，这么更加`简明扼要`。
 - 2.应该通过取`下标操作`来访问数组下标或字典中的键所对应的元素。
 - 3.用字面量语法创建数组或字典时，都是不可变的，而且当其中有值为nil时就会抛异常， 所以一定要保证其中不含nil。



## ④ 多用常量类型, 少用#define预处理指令

预处理过程会把碰到的所有你定义的关键字替换成为你为此关键字指定的值，而且只是替换。而常量类型是指用`static`和`const`修饰的变量，使用起来像是常量一样。下面列举两者：

``` swift
#define ANOATION_DURATION 0.3
static const NSTimeInterval kAnimationDuration = 0.3;
```

开发中根据具体情况来使用修饰符`static`和`const`。`const`修饰的变量不可被修改，将变量常量化。`static`修饰的变量在定义的编译单元中是可见的。编译器每收到一个编译单元就会输出一份`目标文件`(object file)。假如此时变量不加`static`或者加`extern`修饰的话， 编译器会为它创建一个`外部符号`(external symbol)。此时若是另外一个编译单元也声明了同名变量，那么编译器会报错 `duplicate symbol *your var name*`。

所以，若常量局限于某`编译单元`(指实现文件)之内，变量名前加字母`k`；若常量在类之外也可见，则通常以类名为前缀。

``` swift
// EOCAnimatedView.h  
#import <UIKit/UIKit.h> 
 
@interface EOCAnimatedView : UIView  
- (void)animate;  
@end  
 
// EOCAnimatedView.m  
#import "EOCAnimatedView.h"  
 
static const NSTimeInterval kAnimationDuration = 0.3;  
 
@implementation EOCAnimatedView  
- (void)animate {  
    [UIViewanimateWithDuration:kAnimationDuration  
                    animations:^(){  
                         // Perform animations  
                    }];  
}  
@end 
```

static修饰局部变量作用：   
1.让局部变量只初始化一次   
2.局部变量在程序中只有一份内存  
3.并不会改变局部变量的作用域，仅仅是改变了局部变量的生命周期(只到程序结束，这个局部变量才会销毁)  


static修饰全局变量作用：  
1.全局变量的作用域仅限于当前文件   


有时候需要堤外公开某个常量，此类常量需方法`全局符号表`(global symbol table)中，以便可以在编译单元外使用。外部进行使用时，需添加头文件，然后便可使用此常量，用法如下:

``` swift
// in the header file
extern NSString *const EOCStringConstant;

// in the implementation file
NSString *const EOCStringConstant = @"value";
```


 - 1.不要用预处理指令定义常量。这样定义出来的常量不含类型信息， 编译器只会在编译前据此执行查找和替换操作。`const`是在编译中的进行的, 会进行编译检查。使用宏定义时每次都需要创建一份内存, 而`static const`修饰的变量只有一份内存即可。
 - 2.在实现文件中使用`static const`来定义只在编译单元内可见的常量。由于此类常量不在全局符号表中，所以无须为其名称加前缀。
 - 3.在头文件中使用`extern`来声明全局常量，并在相关实现文件中定义其值。这种常量要出现在全局符号表中，所以起名称应加以区别，通常用与本类名做前缀。


## ⑤ 用枚举表示状态、选项、状态码

实现枚举所用的数据类型`取决于编译器`，不过其二进制位(bit)的个数必须能完全表示下枚举编号才行。一个字节含8个二进制位，所以最多能表示可取256种(2^8个)枚举(编号0~255)的枚举变量。

``` swift
enum EOCConnectionState {
	EOCConnectionStateDisconnected,
	EOCConnectionStateConnecting,
	EOCConnectionStateConnected,
};
typedef enum EOCConnectionState EOCConnectionState；
```

上图中的枚举状态共有三种，最大编号是2，所以使用1个字节的char类型即可。

而在上图中的枚举使用时，在没有`typedef`前，还需要如下：

``` swift
enum EOOCConectionState state = EOCConnectionStateDisconnected;
```

所以，在OC中表示状态的枚举一般使用`NS_ENUM`。

还有一种情况，就是定义的枚举项可以任意组合。各项之间就可以通过`按位或操作符`来组合。每个选项均可开启或禁用，每个枚举值对应某一个二进制的开关。就像下面这样：

``` swift
typedef NS_OPTIONS(NSUInteger, UIViewAutoresizing) {
    UIViewAutoresizingNone                 = 0,
    UIViewAutoresizingFlexibleLeftMargin   = 1 << 0,
    UIViewAutoresizingFlexibleWidth        = 1 << 1,
    UIViewAutoresizingFlexibleRightMargin  = 1 << 2,
    UIViewAutoresizingFlexibleTopMargin    = 1 << 3,
    UIViewAutoresizingFlexibleHeight       = 1 << 4,
    UIViewAutoresizingFlexibleBottomMargin = 1 << 5
};
```
其每个选项控制的二进制位表示为：

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-1/20170510225556753.png" width="80%" img/>
</center>

总之，凡是需要以按位或操作符来组合的枚举都应该使用`NS_OPTIONS`来定义枚举类型，若是不需要相互组合，则应使用`NS_ENUM`来定义枚举类型。

``` swift
typedef NS_ENUM(NSUInteger, EOCConnectionState) {
	EOCConnectionStateDisconnected,
	EOCConnectionStateConnecting,
	EOCConnectionStateConnected,
};
typedef NS_OPTIONS(NSUInteger, EOCPermittedDirection) {
	EOCPermittedDirectionUp    = 1 << 0,
	EOCPermittedDirectionLeft  = 1 << 1,
	EOCPermittedDirectionDown  = 1 << 2,
	EOCPermittedDirectionRight = 1 << 3,
};
```

还有关于枚举来定义状态机，一般我们在使用`switch`的时候，习惯加上`default`分支。然而，在结合枚举值使用它的话，是有一些技巧的。如果我们去掉`default`分支，当新加入枚举之后，如果你没有在`switch`分支中处理的话，编译器会报警告，提示开发者来处理这个新状态。

``` swift
typedef NS_ENUM(NSUInteger, EOCConnectionState) {
	EOCConnectionStateDisconnected,
	EOCConnectionStateConnecting,
	EOCConnectionStateConnected,
};
switch (currentState) {
	case：EOCConnectionStateDisconnected：
		// handle code
		break；
	case：EOCConnectionStateConnecting：
		// handle code
		break；
	case：EOCConnectionStateConnected：
		// handle code
		break；
}
```

## 相关资料

[高效 OC开发之熟悉Objective-C](https://wangyanchang21.github.io/2017/Effective-OC%E4%B9%8B%E4%B8%80)  
[高效 OC开发之对象、消息、运行时](https://wangyanchang21.github.io/2017/Effective-OC%E4%B9%8B%E4%BA%8C)  
[高效 OC开发之接口与API设计](https://wangyanchang21.github.io/2017/Effective-OC%E4%B9%8B%E4%B8%89)  
[高效 OC开发之协议与分类](https://wangyanchang21.github.io/2018/Effective-OC%E4%B9%8B%E5%9B%9B)  
[高效 OC开发之内存管理](https://wangyanchang21.github.io/2018/Effective-OC%E4%B9%8B%E4%BA%94)  
[高效 OC开发之Block和GCD](https://wangyanchang21.github.io/2018/Effective-OC%E4%B9%8B%E5%85%AD)  
[高效 OC开发之系统框架](https://wangyanchang21.github.io/2018/Effective-OC%E4%B9%8B%E4%B8%83)  

-------

欢迎指正, [wangyanchang21](https://github.com/wangyanchang21).


