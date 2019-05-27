---
title:  "高效 OC开发之协议与分类"
date:   2018-02-08 17:11:39
categories: [iOS, Effective OC]
tags: [iOS, Effective OC]
---

关于协议，分类，以及扩展的高效使用。

[![Contact](https://img.shields.io/badge/contact-wangyanchang21-green.svg)](https://github.com/wangyanchang21)  

------

- [㉓ 通过委托与数据源协议进行对象间通信](#-通过委托与数据源协议进行对象间通信)
- [㉔ 将类的实现代码分散到便于管理的数个分类之中](#-将类的实现代码分散到便于管理的数个分类之中)
- [㉖ 勿在分类中声明属性](#-勿在分类中声明属性)
- [㉗ 使用“Extension扩展”隐藏实现细节](#-使用extension扩展隐藏实现细节)
- [㉘ 通过协议提供匿名对象](#-通过协议提供匿名对象)  


-------


## ㉓ 通过委托与数据源协议进行对象间通信

### 代理模式/委托模式

对象之间经常需要相互通信, 而通信方式有很多种。 Objective-C开发者广泛使用一种名叫`委托模式`(Delegate pattern)的编程设计模式来实现对象间的通信。

此模式可将数据与业务逻辑解耦。委托协议名通常是在相关类名后面加上 `Delegate` 一词,整个类名采用`驼峰法`来写。这个属性需定义成`weak`,而非 `strong`, 因为两者之间必须为`非拥有关系`(nonowning relationship)。本类中存放委托对象的这个属性要定义为`weak`, 在相关对象销毁时会自动清空。

``` swift
@protocol SubDogDelegate<NSObject>
@optional
- (void)makeNoise;
@end

@interface SubDog : SuperDog
@property (nonatomic, weak) id<SubDogDelegate> delegate;
@end
```

如果要在委托对象上调用可选方法, 那么必须提前使用类型信息查询方法(参见第14条)判断这个委托对象能否响应相关选择子, 以及委托是否遵循了当前的委托方法。所以应该这样写:

``` swift
if ([_delegate conformsToProtocol:@protocol(SubDogDelegate)] && [_delegate respondsToSelector:@selector(makeNoise)]) {
	[_delegate makeNoise];
}
```

也可以用协议定义一套接口,令某类经由该接口获取其所需的数据。委托模式的这用法旨在向类提供数据,故而又称`数据源模式`(Data Source pattern)。在此模式中,信息从数据源(Data Source)流向类(Class);而在常规的委托模式中,信息则从类流向受委托者(Delegate)。

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-4/20180206171639202.png" width="70%" img/>
</center>


将数据源协议与委托协议分离, 能使接口更加清晰,因为这两部分的逻辑代码也分开了。另外,`数据源`与`受委托者`可以是两个不同的对象。然而一般情况下, 都用同一个对象来扮演这两种角色。


### 效率优化

虽然上面方法中的判断很简单, 但在某些情况下可能会多次调用, 如果每次都检查委托对象是否能响应此选择子, 那就显得多余了。鉴于此, 我们可以把委托对象能否响应某个协议方法这一信息缓存起来, 以优化程序效率。

将方法响应能力缓存起来的最佳途径是使用`位段`(bitfield)数据类型, 而新增的这个实例变量是个C语言结构体, 其中含有三个位段, 每个位段都与 `delegate`所遵从的协议中某个可选方法相对应。在下面例子中, 可以像下面这样查询并设置结构体中的位段:

``` swift
@protocol SubDogDelegate<NSObject>
@optional
- (void)didReceiveData;
- (void)didFilwithError;
- (void)didUpdateProgress;
@end

@interface SubDog : SuperDog
@property (nonatomic, weak) id<SubDogDelegate> delegate;
@end
```


``` swift
@interface SubDog () {
    struct {
        unsigned int didReceiveData : 1;
        unsigned int didFilwithError : 1;
        unsigned int didUpdateProgress : 1;
    } _delegateFlags;
}

@end

@implementation SubDog

- (void)setDelegate:(id<SubDogDelegate>)delegate {
    _delegate = delegate;
    // 设置delegate时, 设置不同位段的flag缓存
    _delegateFlags.didReceiveData = [_delegate conformsToProtocol:@protocol(SubDogDelegate)] && [_delegate respondsToSelector:@selector(didReceiveData)];
    _delegateFlags.didFilwithError = [_delegate conformsToProtocol:@protocol(SubDogDelegate)] && [_delegate respondsToSelector:@selector(didFilwithError)];
    _delegateFlags.didUpdateProgress = [_delegate conformsToProtocol:@protocol(SubDogDelegate)] && [_delegate respondsToSelector:@selector(didUpdateProgress)];
}

- (void)requestResultMethod {
    // 不同的代理方法, 使用不同位段的flag缓存
    if (_delegateFlags.didReceiveData) {
        [_delegate didReceiveData];
    } else if (_delegateFlags.didFilwithError) {
        [_delegate didFilwithError];
    } else if (_delegateFlags.didUpdateProgress) {
        [_delegate didUpdateProgress];
    }
}

@end
```

在相关方法要调用很多次时, 值得进行这种优化。而是否需要优化, 则应依照具体代码来定。这就需要分析代码性能, 并找出瓶颈, 若发现执行速度需要改进, 则可使用此技巧。如果要频繁通过数据源协议从数据源中获取多份相互独立的数据, 那么这项优化技术极有可能会提高程序效率。


### 总结

 - 1.委托模式为对象提供了一套接口, 使其可由此将相关事件告知其他对象。
 - 2.将委托对象应该支持的接口定义成协议, 在协议中把可能需要处理的事件定义成方法。当某对象需要从另外一个对象中获取数据时, 可以使用委托模式。这种情境下, 该模式亦称`数据源协议`(data source protocal)。
 - 3.若有必要, 可实现含有位段的结构体, 将委托对象是否能响应相关协议方法这一信息缓存至其中。


## ㉔ 将类的实现代码分散到便于管理的数个分类之中

在实现某些类时, 假如类中的方法代码非常多, 那么该类的实现文件就十分臃肿。如果还向类中继续添加方法的话, 那么源代码文件就会越来越大, 变得难于管理。在此情况下,可以通过 Objective-C的`分类`机制, 把类代码按逻辑划入几个分区中, 把类分成几个不同的部分, 这对开发与调试都有好处。如下:

``` swift
@interface Person : NSObject
@property (nonatomic,copy) NSString *name;
@property (nonatomic,readonly) NSArray *friends;
@property (nonatomic,assign) int age;
@end

@interface Person (FriendShip)
- (void)addFriend:(Person *)person;
- (void)removeFriend:(Person *)person;
@end

@interface Person (Work)
- (void)performDaysWork;
- (void)takeVacationFromWork;
@end

@interface Person (Play)
- (void)playPingPong;
- (void)playFootball;
- (void)sing;
- (void)run;
@end
```

使用分类机制之后, 依然可以把整个类都定义在一个接口文件中, 并将其代码写在一个实现文件里。可是, 随着分类数量增加, 当前这份实现文件很快就膨胀得无法管理了。此时可以把每个分类提取到各自的文件中去。以 Person为例, 可以按照其分类将代码拆分成下列几个文件: 
 
 - 1.Person+Friendship(h/m)
 - 2.Person+Work(h/m)
 - 3.Person+Play(h/m)

使用分类机制之后, 如果想用分类中的方法, 那么要记得在引入`Person.h`时一并引入分类的头文件。虽然稍微有点麻烦, 不过分类仍然是一种管理代码的好办法。而且, 这样做还有一个好处, 就是便于调试。根据回溯信息中的分类名称, 很容易就能精确定位到类中的方法所属的功能区。

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-4/20180207110456491.png" img/>
</center>

这对于某些`私有方法`来说更是极为有用, 可以创建名为 Private 的分类, 把这种方法全都放在里面。这个分类里的方法一般只会在类或框架内部使用, 而无须对外公布。在编写准备分享给其他开发者使用的程序库时, 可以考虑创建 Private分类。可以非常灵活的选择随不随程序库一并公开。

### 总结

 - 1.使用分类机制把类的实现代码划分成易于管理的小块, 且易于调试。
 - 2.将应该视为`私有方法`归入名叫 Private的分类中,以隐藏实现细节。


## ㉕ 总是为第三方类的分类名称加前缀

分类机制通常用于向无源码的既有类中新增功能。将分类方法加入类中这一操作是在运行期系统加载分类时完成的。如果类中本来就有此方法,而分类又实现了一次,那么分类中的方法会覆盖原来那一份实现代码。多次覆盖的结果以最后一个分类为准。

以命名空间来区别各个分类的名称与其中所定义的方法。给相关名称都加上某个共用的前缀。也与给分类所加的前缀, 当然这与给类名加前缀(参见第15条)时所应考虑的因素相似。

``` swift
@interface Nsstring (ABC_HTTP)

// Encode a string with URL encoding
-(Nsstring*)abc_urlEncodedstring:

// Decode a URL encoded string
-(Nsstring*)abc_urlDecodedstring;

@end
```

这样做也能避免类的开发者以后在更新该类时所添加的方法与你在分类中添加的方法重名。否则, 很可能当你所写的代码所覆盖系统或者其他三方库的原犯法后, 则会令对象内的数据互不一致,从而造成难于查找的bug。

### 总结

 - 1.向第三方类中添加分类时,总应给其名称加上你专用的前缀
 - 2.向第三方类中添加分类时,总应给其中的方法名加上你专用的前缀。


## ㉖ 勿在分类中声明属性

属性是封装数据的方式(参见第6条)。尽管从技术上说,分类里也可以声明属性,但这种做法还是要尽量避免。原因在于, 除`Extension扩展`之外的其他分类无法把实现属性所需的实例变量合成出来。例如, 你在分类中添加一个`name` 的属性时, 编译器应该会提示你:

> Property 'name' requires method 'name' to be defined - use @dynamic or provide a method implementation in this category


意思是说此分类无法合成与 `name`属性相关的实例变量, 所以开发者需要在分类中为该属性实现存取方法。当然你可以把存取方法声明为`@dynamic`, 等到运行期再提供, 使用消息转发机制(参见第12条)在运行期拦截方法调用, 并提供其实现。但是一般我们通过关联对象(参见第10条)就能解决这个问题:

``` swift
// .h
@property (nonamic, assign) BOOL canShowToast;

// .m
- (BOOL)canShowToast {
    BOOL canShowToast = [objc_getAssociatedObject(self, kSomeKey) boolValue];
    return canShowToast;
}

- (void)setCanShowToast:(BOOL)canShowToast {
    objc_setAssociatedObject(self, @selector(canShowToast), kSomeKey, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
```

这样做可行,但不太理想。要把相似的代码写很多遍, 而且在内存管理问题上容易出错, 因为在为属性实现存取方法时, 经常会忘记遵从其内存管理语义。比方说, 你可能通过属性特质(attribute)修改了某个属性的内存管理语义。而此时还要记得, 在设置方法中也得修改设置关联对象时所用的内存管理语义才行。

所以, 正确做法是把所有属性都定义在主接口里, 而且这样要比定义在分类里清晰得多。因为对于分类机制, 其目的在于扩展类的功能, 而非封装数据。

### 总结

 - 1.把封装数据所用的全部属性都定义在主接口里。
 - 2.在`Extension扩展`之外的其他分类中, 可以定义存取方法, 但尽量不要定义属性。


## ㉗ 使用“Extension扩展”隐藏实现细节

Objective-C动态消息系统(参见第11条)的工作方式决定了其不可能实现真正的私有方法或私有实例变量。

OC中的`Extension`, 即扩展, 也被称为`class-continuation`分类。这是唯一能声明实例变量的分类, 而且此分类没有特定的实现文件, 其中的方法都应该定义在类的主实现文件里。

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-4/20180208161629311.png" img/>
</center>

公共接口里本来就能定义实例变量。不过, 把它们定义在`Extension扩展`或`实现块`中可以将其隐藏起来, 只供本类使用。而且还可以在`Extension扩展`中服从协议, 这样也不会对外界暴露。

``` swift
// .m
@interface ViewController () {
    NSInteger _age;
}
@property (nonatomic, copy) NSString *str;
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    
}
@end
```

`Extension扩展` 还有一种合理用法, 就是将 public接口中声明为`只读`的属性扩展为`可读写`, 以便在类的内部设置其值。这样, 封装在类中的数据就由实例本身来控制, 而外部代码则无法修改其值。

``` swift
// .h
@interface ViewController : UIViewController
@property (nonatomic, copy, readonly) NSString *string;
@end

// .m
@interface ViewController ()
@property (nonatomic, copy, readwrite) NSString *string;
@end

@implementation ViewController
@end
```

只会在类的实现代码中用到的私有方法也可以声明在`Extension扩展`中。

``` swift
@interface ViewController ()
- (void)private_someMethod;
@end
```

这样做可以把类里所含的相关方法都统一描述于此, 使类的代码更易读懂。若对象所遵从的协议只应视为私有,则可在`Extension扩展`中声明。

### 总结

 - 1.通过`Extension扩展`向类中新增实例变量。
 - 2.如果某属性在主接口中声明为`只读`, 而类的内部又要用设置方法修改此属性, 那么就在`Extension扩展`中将其扩展为`可读写`。
 - 3.把私有方法的原型声明在`Extension扩展`里面。
 - 4.若想使类所遵循的协议不为人所知, 则可于`Extension扩展`中声明。


## ㉘ 通过协议提供匿名对象

我们可以用协议把自己所写的API之中的实现细节隐藏起来, 将返回的对象设计为遵从此协议的纯`id类型`。这样的话, 想要隐藏的类名就不会出现在API之中了。若是接口背后有多个不同的实现类,而你又不想指明具体使用哪个类, 那么可以考虑使用这个方法。

数据库连接(database connection)的程序也用这个思路，以匿名对象来表示从另一个库中返回的对象。对于处理连接哪个类，你也许不想让万人知道。如果没有办法令其继承字同一个基类，那么就得返回对下你跟遵从此协议：

``` swift
@protocol EOCDatabaseConnection
- (void)connect;
- (void)disconnect;
- (void)isConnected;
- (NSArray*)performQuery:(NSString *)query;
@end;
```

然后，就可以用`数据库处理器`单例来提供数据库连接了。这个单例的接口可以写成：

``` swift
@protocol EOCDatabaseConnection
@interface EOCDatabaseManger:NSObject
+ (id)sharedInstance;
- (id<EOCDatabaseConnection>) connectionWithIdentifier:(NSString *)identifier;
@end;
```

### 总结

 - 1.协议可在某种程度上提供匿名类型。具体的对象类型可以淡化成遵从某协议的`id类型`，协议里规定了对象所应实现的方法。
 - 2.使用匿名对象来隐藏类型名称(或类名)
 - 3.如果具体类型不重要，重要的是对象能够响应特定方法，那么可以是使用匿名对象来表示。




相关资料：   
[高效 OC开发之熟悉Objective-C](https://wangyanchang21.github.io/2017/Effective-OC%E4%B9%8B%E4%B8%80)  
[高效 OC开发之对象、消息、运行时](https://wangyanchang21.github.io/2017/Effective-OC%E4%B9%8B%E4%BA%8C)  
[高效 OC开发之接口与API设计](https://wangyanchang21.github.io/2017/Effective-OC%E4%B9%8B%E4%B8%89)  
[高效 OC开发之协议与分类](https://wangyanchang21.github.io/2018/Effective-OC%E4%B9%8B%E5%9B%9B)  
[高效 OC开发之内存管理](https://wangyanchang21.github.io/2018/Effective-OC%E4%B9%8B%E4%BA%94)  
[高效 OC开发之Block和GCD](https://wangyanchang21.github.io/2018/Effective-OC%E4%B9%8B%E5%85%AD)  
[高效 OC开发之系统框架](https://wangyanchang21.github.io/2018/Effective-OC%E4%B9%8B%E4%B8%83)  

-------

欢迎指正, [wangyanchang21](https://github.com/wangyanchang21).


