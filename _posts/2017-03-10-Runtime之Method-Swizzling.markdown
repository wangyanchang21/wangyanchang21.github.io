---
title:  "Runtime黑魔法之Method Swizzling"
date:   2017-03-10 18:03:23
categories: [iOS, runtime]
tags: [iOS, runtime]
---

runtime中通过 swizzle method实现方法交换，以解决统一派生基类的不足的问题。

[![Contact](https://img.shields.io/badge/contact-wangyanchang21-green.svg)](https://github.com/wangyanchang21)

------

- [目的](#目的)
	- [统一派生基类的不足](#统一派生基类的不足)
	- [基类派生的替代方法](#基类派生的替代方法)
- [AOP编程思想-Method Swizzling](#aop编程思想-method-swizzling)
	- [Method Swizzling原理](#method-swizzling原理)
	- [Method Swizzling使用](#method-swizzling使用)
	- [代码分析](#代码分析)
	- [注意](#注意)
	- [swizzle方法封装](#swizzle方法封装)

------

## 目的

我们要在所有页面要进行一个统一的设置或者添加一个一样的方法时，最直接的就是每个页面都code一次, 复制粘贴...但是这样太low了。  如果不这样呢, 很多人采取的方法就是建一个基类, 比如创建一个基于UIViewController的基类DCViewController, 在这个基类中写下统一的设置或者方法。 但是我个人认为, 这样的方式还是有不妥的地方. 或者使用category, 在controller中进行调用。

那到底怎么才是上上策呢, 这就是今天我要说的话题。

### 统一派生基类的不足

1.集成成本 
对于业务层存在的所有父类来说，它们是很容易跟项目中的其他代码纠缠不清的，这使得业务方开发时遇到一个两难问题：要么把所有依赖全部搞定，然后基于App环境下开发Demo，要么就是自己Demo写好之后，按照环境要求改代码。这里的两难问题都会带来成本，都会影响业务方的迭代进度。 

2.新来的业务工程师有的时候不见得都记得每一个ViewController都必须要派生自DCViewController而不是直接的UIViewController, 所以定制性很差。以后有新人加入之后，都要嘱咐其继承自这个基类，所以这种方式并不可取。比如说：所有的ViewController都必须继承自DCViewController。 

3.架构维护难度提升

### 基类派生的替代方法

建议使用AOP来代替派生解决此类问题, 达到的效果也是如下:    

- 1.业务方可以不用通过继承的方法，然后框架能够做到对ViewController的统一配置。    
- 2.业务方即使脱离框架环境，不需要修改任何代码也能够跑完代码。业务方的ViewController一旦丢入框架环境，不需要修改任何代码，框架就能够起到它应该起的作用。   

其实就是要实现不通过业务代码上对框架的主动迎合，使得业务能够被框架感知这样的功能。细化下来就是两个问题，框架要能够拦截到ViewController的生命周期，另一个问题就是，拦截的定义时机。 

对于方法拦截，很容易想到`Method Swizzling`，那么我们可以写一个实例，在App启动的时候添加针对UIViewController的方法拦截，这是一种做法。还有另一种做法就是，使用NSObject的load函数，在应用启动时自动监听。使用后者的好处在于，这个模块只要被项目包含，就能够发挥作用，不需要在项目里面添加任何代码。

然后另外一个要考虑的事情就是，原有的DCViewController（所谓的父类）也是会提供额外方法方便子类使用的，`Method Swizzling`只支持针对现有方法的操作，拓展方法的话，需要使用分类进行拓展。 

我本人不赞成Category的过度使用，但鉴于Category是最典型的化继承为组合的手段，在这个场景下还是适合使用的。还有的就是，关于`Method Swizzling`手段实现方法拦截，业界也已经有了现成的开源库：[Aspects](https://github.com/steipete/Aspects)，我们可以直接拿来使用。

## AOP编程思想-Method Swizzling

### Method Swizzling原理

Method Swizzing是发生在运行时的，主要用于在运行时将两个Method进行交换，我们可以将`Method Swizzling`代码写到任何地方，但是只有在这段`Method Swizzling`代码执行完毕之后互换才起作用。

而且`Method Swizzling`也是iOS中AOP(面相切面编程)的一种实现方式，我们可以利用苹果这一特性来实现AOP编程。

首先，让我们通过两张图片来了解一下`Method Swizzling`的实现原理:

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/methodswizzle/20170310161902367.png" width="70%" img/>
</center>

dispath talbe的概念:   

- In computer science, a dispatch table is a table of pointers to functions or methods. Use of such a table is a common technique when implementing late binding in object-oriented programming.

在Objective-C中调用一个方法，其实是向一个对象发送消息，查找消息的唯一依据是`SEL`的名字。在每个类中都有一个`Dispatch Table`，这个`Dispatch Table`本质是将类中的`SEL`和`IMP`(指向这个方法实现的函数指针)进行对应。第一张图就是体现了`SEL`和这个方法实现的函数指针的映射关系。

在OC语言的runtime特性中，调用一个对象的方法就是给这个对象发送消息。是通过查找接收消息对象的dispath table，从中查找对应的`SEL`，这个`SEL`对应着一个`IMP`(一个`IMP`可以对应多个`SEL`)，通过这个`IMP`找到对应的方法调用。


<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/methodswizzle/20170310161911945.png" width="70%" img/>
</center>

而`Method Swizzling`就是对 dispath table进行了操作，让`SEL`对应另一个`IMP`。利用Objective-C的动态特性，改变其映射, 它可以使得在运行时通过改变 `SEL` 在类的消息分发列表中的映射从而改变方法的调用。

第二图中SELC原本对应着IMPc，但是为了更方便的实现特定业务需求，我们在图二中添加了SELN和IMPn，并且让SELC指向了IMPn，而SELN则指向了IMPc，这样就实现了`方法互换`。但需要注意的是, `Method Swizzling`只能改变方法的映射, 而不是改变方法的实现。


### Method Swizzling使用

创建UIViewController的分类UIViewController+Swizzling

``` swift
@implementation UIViewController (Swizzling)

+ (void)load {
    [super load];
    
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        
		SEL originalSelector = @selector(viewDidLoad);
		SEL swizzleSelector = @selector(dc_swizzlingViewDidLoad);
        
        Method originalMethod = class_getInstanceMethod([self class], originalSelector);
        Method swizzleMethod = class_getInstanceMethod([self class], swizzleSelector);
        
        BOOL didAddMethod = class_addMethod([self class],
                                            originalSelector,
                                            method_getImplementation(swizzleMethod),
                                            method_getTypeEncoding(swizzleMethod));
        
        if (didAddMethod) {
            class_replaceMethod([self class],
                                swizzleSelector,
                                method_getImplementation(originalMethod),
                                method_getTypeEncoding(originalMethod));
        }else{
            method_exchangeImplementations(originalMethod, swizzleMethod);
        }
    });
}
// 和viewDidLoad方法进行交换的方法
- (void)dc_swizzlingViewDidLoad {
    NSString *str = [NSString stringWithFormat:@"%@", self.class];
    
    // 我们在这里加一个判断，将系统的UIViewController的对象剔除掉
    if(![str containsString:@"UI"]){
        NSLog(@"swizzlingViewDidLoad___统计打点 : %@", self.class);
    }
    
    //这里使用dc_swizzlingViewDidLoad是不会产生递归调用的, 因为方法已经互换, 所以这里调用的是其实是viewDidLoad
    [self dc_swizzlingViewDidLoad];
}

- (void)customBaseMethod
{
    NSLog(@"custom base method has invoked");
}

@end
```

### 代码分析

#### 概念

1.Selectors/Methods/Implementations

在 Objective-C 的运行时中，`selectors`, `methods`, `implementations` 指代了不同概念。下面是三个概念的官方说明:

- Selector（typedef struct objc_selector *SEL）:在运行时 Selectors 用来代表一个方法的名字。Selector 是一个在运行时被注册（或映射）的C类型字符串。Selector由编译器产生并且在当类被加载进内存时由运行时自动进行名字和实现的映射。
- Method（typedef struct objc_method *Method）:方法是一个不透明的用来代表一个方法的定义的类型。
- Implementation（typedef id (*IMP)(id, SEL,...)）:这个数据类型指向一个方法的实现的最开始的地方。该方法为当前CPU架构使用标准的C方法调用来实现。该方法的第一个参数指向调用方法的自身（即内存中类的实例对象，若是调用类方法，该指针则是指向元类对象metaclass）。第二个参数是这个方法的名字selector，该方法的真正参数紧随其后。

总结一下, selector, method, implementation 这三个概念之间关系的最好方式是：在运行时，类（Class）维护了一个消息分发列表来解决消息的正确发送。每一个消息列表的入口是一个方法（Method），这个方法映射了一对键值对，其中键值是这个方法的名字 selector（SEL），值是指向这个方法实现的函数指针 implementation（IMP）。 `Method Swizzling` 修改了类的消息分发列表使得已经存在的 selector 映射了另一个实现 implementation，同时重命名了原生方法的实现为一个新的 selector。

2.class_getInstanceMethod()

返回该类指定的实例方法, 通过`class_getInstanceMethod()`函数从当前类的`method list`获取实例方法，如果是类方法就使用`class_getClassMethod()`函数获取。

``` swift
/** 
* Returns a specified instance method for a given class.
* 
* @param cls The class you want to inspect.
* @param name The selector of the method you want to retrieve.
* 
* @return The method that corresponds to the implementation of the selector specified by 
*  \e name for the class specified by \e cls, or \c NULL if the specified class or its 
*  superclasses do not contain an instance method with the specified selector.
*
* @note This function searches superclasses for implementations, whereas \c class_copyMethodList does not.
*/
```

2.class_addMethod()

我们在这里使用`class_addMethod()`, 对类添加一个交换后对应的方法, 检查当前类有没有添加过此方法

``` swift
/** 
* Adds a new method to a class with a given name and implementation.
* 
* @param cls The class to which to add a method.
* @param name A selector that specifies the name of the method being added.
* @param imp A function which is the implementation of the new method. The function must take at least two arguments—self and _cmd.
* @param types An array of characters that describe the types of the arguments to the method. 
* 
* @return YES if the method was added successfully, otherwise NO 
*  (for example, the class already contains a method implementation with that name).
*
* @note class_addMethod will add an override of a superclass's implementation, 
*  but will not replace an existing implementation in this class. 
*  To change an existing implementation, use method_setImplementation.
*/
```

3.class_replaceMethod()

在上面的代码中逻辑上是这样的, 如果`didAddMethod`返回YES, 即方法添加成功(这里添加的方法名是originalSelector, IMP是swizzleMethod), 所以需要将方法名为swizzleMethod, IMP为originalMethod的方法进行替换, 这样就同样达到了exchang的目的, 更加严谨. `class_replaceMethod`方法调用的时候, 如果没有当前方法, 系统会自动调用`class_addMethod`进行添加. 所以一定能够进行replace.

``` swift
/** 
* Replaces the implementation of a method for a given class.
* 
* @param cls The class you want to modify.
* @param name A selector that identifies the method whose implementation you want to replace.
* @param imp The new implementation for the method identified by name for the class identified by cls.
* @param types An array of characters that describe the types of the arguments to the method. 
*  Since the function must take at least two arguments—self and _cmd, the second and third characters
*  must be “@:” (the first character is the return type).
* 
* @return The previous implementation of the method identified by \e name for the class identified by \e cls.
* 
* @note This function behaves in two different ways:
*  - If the method identified by \e name does not yet exist, it is added as if \c class_addMethod were called. 
*    The type encoding specified by \e types is used as given.
*  - If the method identified by \e name does exist, its \c IMP is replaced as if \c method_setImplementation were called.
*    The type encoding specified by \e types is ignored.
*/
```

4.method_exchangeImplementations()

在上面的代码中逻辑上是这样的, 如果`didAddMethod`返回NO, 即方法添加失败, 说明originalSelector方法名字已经存在, 可以直接使用`method_exchangeImplementations`进行交换.

``` swift
/** 
* Exchanges the implementations of two methods.
* 
* @param m1 Method to exchange with second method.
* @param m2 Method to exchange with first method.
* 
* @note This is an atomic version of the following:
*  \code 
*  IMP imp1 = method_getImplementation(m1);
*  IMP imp2 = method_getImplementation(m2);
*  method_setImplementation(m1, imp2);
*  method_setImplementation(m2, imp1);
*  \endcode
*/
```

#### +load vs +initialize

swizzling应该只在`+load`中完成。 在 Objective-C 的运行时中，每个类有两个方法都会自动调用。`+load` 是在一个类被初始装载时调用，`+initialize` 是在应用第一次调用该类的类方法或实例方法前调用的。两个方法都是可选的，并且只有在方法被实现的情况下才会被调用。

#### dispatch_once

swizzling 应该只在 `dispatch_once` 中完成, 由于 swizzling 改变了全局的状态，所以我们需要确保每个预防措施在运行时都是可用的。原子操作就是这样一个用于确保代码只会被执行一次的预防措施，就算是在不同的线程中也能确保代码只执行一次。`Grand Central Dispatch` 的 `dispatch_once` 满足了所需要的需求，并且应该被当做使用 swizzling 的初始化单例方法的标准。

#### 调用 _cmd

在上面的代码中逻辑上是这样的, 在swizzling交换后的方法`dc_swizzlingViewDidLoad`中又调用了[self dc_swizzlingViewDidLoad], 这样不会产生递归调用吗? 答案是, 这里使用`dc_swizzlingViewDidLoad`是不会产生递归调用的, 因为方法已经互换, 所以这里调用的是其实是已经交换后的viewDidLoad.

### 注意

- 在交换方法实现后记得要调用原生方法的实现（除非你非常确定可以不用调用原生方法的实现）：APIs 提供了输入输出的规则，而在输入输出中间的方法实现就是一个看不见的黑盒。交换了方法实现并且一些回调方法不会调用原生方法的实现这可能会造成底层实现的崩溃。
- 避免冲突：为分类的方法加前缀，一定要确保调用了原生方法的所有地方不会因为你交换了方法的实现而出现意想不到的结果。
- 注意[super load], 如果你为一个父类和一个子类同时写了一个swizzling method, 方法都是交换的是viewDidLoad方法, swizzling后的方法名字也相同的话,而且子类的load中还调用了[super load] . 这样的话, 在子类加载时, 会在成父类的swizzling方法递归调用, 为什么呢, 等有时间了, 我在补充代码说明吧
- 理解实现原理：只是简单的拷贝粘贴交换方法实现的代码而不去理解实现原理很可能会让 App 产生不可思议的事情。阅读 [Objective-C Runtime Reference](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/ObjCRuntimeRef/index.html#//apple_ref/c/func/method_getImplementation) 并且浏览 能够让你更好理解实现原理。
- swizzling method 很方便, 很强大, 但也可能是杀死你的那把刀. 有利有弊, 严谨的使用, 以免上线那天泪奔不止.


### swizzle方法封装

你可以将此方法封装起来为了让你更方便的使用, 当然也不是必须一模一样地按照上面的代码来进行实现. 比如下面在NSObject分类中的实现, 方便且不失严谨:

``` swift
+ (BOOL)swizzleMethod:(SEL)originSelector withMethod:(SEL)swizzleSelector {
    Method originMethod = class_getInstanceMethod(self, originSelector);
    if (!originMethod) {
        return NO;
    }

    Method swizzleMethod = class_getInstanceMethod(self, swizzleSelector);
    if (!swizzleMethod) {
        return NO;
    }

    class_addMethod(self,
                    originSelector,
                    class_getMethodImplementation(self, originSelector),
                    method_getTypeEncoding(originMethod));

    class_addMethod(self,
                    swizzleSelector,
                    class_getMethodImplementation(self, swizzleSelector),
                    method_getTypeEncoding(swizzleMethod));

    method_exchangeImplementations(class_getInstanceMethod(self, originSelector), class_getInstanceMethod(self, swizzleSelector));

    return YES;
}
```

-------

欢迎指正, [wangyanchang21](https://github.com/wangyanchang21).


