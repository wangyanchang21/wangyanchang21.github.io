---
title:  "高效 OC开发之接口与API设计"
date:   2017-11-05 13:02:38
categories: [iOS, Effective OC]
tags: [iOS, Effective OC]
---

Objective-C的命名规则及命名空间，私有方法的处理。另外，还有readOnly，NSError，NSCopying协议的高效使用。

[![Contact](https://img.shields.io/badge/contact-wangyanchang21-green.svg)](https://github.com/wangyanchang21)  

------

- [⑮ 用前缀避免命名空间冲突](#-用前缀避免命名空间冲突)
- [⑯ 提供“全能初始化方法”](#-提供全能初始化方法)
- [⑰ 实现description方法](#-实现description方法)
- [⑱ 尽量使用不可变对象](#-尽量使用不可变对象)
- [⑲ 使用清晰而协调的命名方式](#-使用清晰而协调的命名方式)
- [⑳ 为私有方法名加前缀](#-为私有方法名加前缀)
- [㉑ 理解Objective-C 错误模型](#-理解objective-c-错误模型)
- [㉒ 理解NSCopying协议](#-理解nscopying协议)
- [相关资料](#相关资料)

-------


## ⑮ 用前缀避免命名空间冲突

如果发生`命名冲突`(naming clash)，那么应用程序的链接过程就会出错，因为出现了重复符号。

Apple宣称其保留使用所有`两字母前缀`(two-letter prefix)的权利，所以你自己选用的前缀应该是`三个字母`的。

不仅是命名，应用程序中的所有名称都应加前缀。如果要为既有类新增`分类`(category)，那么一定要给分类及分类中的方法加上前缀。另外一个容易引发命名冲突的地方，就是类的实现文件中所用的纯C函数及全局变量，这个问题必须要注意。在编译好的目标文件中，这些名称是要算作`顶级符号`(top-level symbol)的。

### 总结

 - 1.选择与你的公司、应用程序或两者皆有关联之名称作为类名的前缀，并在所有代码中均使用这一前缀。
 - 2.若自己所开发的程序库中用到了第三方库，则应为其中的名称加上前缀。


## ⑯ 提供“全能初始化方法”

可为对象提供必要信息以便其能完成工作的初始化方法叫做`全能初始化方法`(designated initializer)。就像下面的这个方法:

``` swift
- (id)initWithWidth:(float)width andHeight:(float)height {
	if (self = [super init]){
		_width = width;
		_height = height;
	}
	return self;
}
```

在类中提供一个全能初始化方法，其他初始化方法均应调用此方法。而且, 每个子类的全能初始化方法都应该调用其超类的对应方法，并逐层向上。但是, 如果要调用`[[SomeClass alloc] init]`的话, 就需要处理一下了。一般的, 两种方式, 通过全能初始化方法设置默认值或者抛异常。

``` swift
- (id)init {
	return [self initWithWidth:10 andHeight:20];
}
```

或者:

``` swift
- (id)init {
	@throw [NSException exceptionWithName:NSInternalInconsistencyException reason:@"Must use initWithWidth:andHeight: instead" userInfo:nil];
}
```

有时候需要编写多个全能初始化方法, 比如说某对象的实例有两种完全不同的创建方式, 必须分开。下面以`NSCoding协议`为例, `NSCoding协议`定义了下面的初始化方法, 遵从该协议都应该实现:

``` swift
- (id)initWithCoder:(NSCoder *)decoder;
```

而且, 此方法必须在父类和其所有子类中必须实现。

### 总结

 - 1.在类中提供一个全能初始化方法，并于文档里指明。其他初始化方法均应调用此方法。
 - 2.若全能初始化方法与超类不同，则需覆写超类中的对应方法。
 - 3.如果超类的初始化方法不适用于子类，那么应该覆写这个超类方法，并在其中抛出异常。


## ⑰ 实现description方法

在调用NSLog(@"object = %@",object); 其实是调用了对象的`description`方法。在我们自定义类中，打印输出信息有可能是这种object = `<SomeClass: 0x7f9d8978679>`，这的话就需要我们重写`description`方法，让它返回我们需要的一些有自定义的信息。

`debugDescription`方法是开发者在调试器中以控制台命令打印对象时才调用的(控制台中输入po object)。而且在默认是直接调用`description`方法, 除非你重写 `debugDescription`方法。

### 总结

 - 1.实现`description`方法返回一个有意义的字符串，用以描述该实例。
 - 2.若想在调试时打印出更详尽的对象描述信息，则应实现`debugDescription`方法。


## ⑱ 尽量使用不可变对象

设计类的时候，用属性来封装数据，在用属性时，当外部不需要修改其内容时, 可将其声明为`只读`并暴露出来。所以, 尽量避免暴露属性, 如果需要暴露尽量为只读, 其次再考虑读写。

虽然属性对外设置成`readonly`了，但是外部仍能通过`键值编码`(Key-Value Coding，KVC)技术设置这些属性值。[object setValue:@"abc" forKey:@"name"] ，这样子可以修改name这个属性，`KVC` 会在类中查找 `setName:`方法来修改属性值。

还可以通过类型信息查询功能，查出属性所对应的实例变量在内存中的偏移量，从此来人为设置这个实例变量的值。不过, 这样做等于违规地绕过了类所提供的API, 要是开发者使用这种杂技代码(hock)的话, 那么得自己来应对可能出现的问题了。

### 总结

 - 1.尽量创建不可变的对象。
 - 2.若某属性仅可于对象内部修改，则在 `Extension`(即扩展) 中将其由`readonly`属性扩展成`readwrite`属性。
 - 3.不要把`可变的collection`作为公开属性，而应提供相关方法，以此修改对象中的可变collection。

## ⑲ 使用清晰而协调的命名方式

按照OC自身的命名规则来说, 方法名和变量名采用`驼峰式大小写命名法`：以小写字母开头，其后每个单词首字母大写。类名也采用驼峰式命名法，不过其首字母需要大写，通常还会加两三个字母作前缀。而且OC语言中在命名上跟其他语言不同, 它是很清晰的, 不管是类名, 变量还是方法名都能很明确的读懂, 所有有时OC的命名会很长。

### 方法命名

把方法名起的稍微长一点，可以保证其能准确传达出方法所执行的任务，但是也不能累赘，尽量言简意赅。清晰的方法名从左至右读起来好似一篇文章，易于维护，他人也更加易懂。

举个例子作对比:

``` swift
// 第一种
- (EOCRectangle *)unionRectangle:(EOCRectangle *)rectangle
- (float)area

// 第二种
- (EOCRectangle *)union:(EOCRectangle *)rectangle // Unclear
- (float)calculateTheArea // Too verbose
```

1.如果方法的返回值是新创建的，那么方法名的首个词应该是返回值的类型，除非前面还有修饰语，例如`localizedString`。属性的存取方法不遵循这种命名方式，因为一般认为这些方法不会创建新对象，即便有时返回内部对象的一份拷贝，我们也认为那相当于原有的对象。这些存取方法应该按照其所对应的属性来命名。   
2.应该把表示参数类型的名词放在参数前面。   
3.如果方法要在当前对象上执行操作，那么就应该包含动词；若执行操作时还需要参数，则应该在动词后面加上一个或多个名字。   
4.不要使用str这种简称，应该使用string这样的全称, 当然也不要过于冗长。   
5.Boolean 属性应加`is`前缀。如果某方法返回非属性的Boolean 值，那么应该根据其功能，选用`has`或`is`当前缀。   
6.将get 这个前缀留给那些借由`输出参数`来保存返回值的方法，比如说，把返回值填充到`C语言式数组`里的那种方法就可以使用这个词做前缀。   

### 类与协议的命名

应该为类与协议的名称加上前缀，以避免命名空间冲突。

命名应该协调一致，从其他框架继承子类，务必遵循其命名惯例。比如, UIView子类末尾必须是View，委托协议末尾必须是Delegate。


### 总结

 - 1.起名时应遵从标准的Objective-C 命名规范，这样子创建出来的接口更容易为开发者所理解。
 - 2.方法名要言简意赅，从左至右读起来要像个日常用语中的句子才好。
 - 3.方法名里不要使用缩略后的类型名称。
 - 4.给方法起名时第一要务就是确保其风格与你自己的代码或所有集成的框架相符。


## ⑳ 为私有方法名加前缀

编写类的实现代码时，经常要写一些只在内部使用的方法, 即私有方法。笔者建议, 应该为私有方法的名称加上某些前缀，这有助于调试，因为据此很容易就能把公共方法和私有方法区别开。

苹果公司喜欢单用一个下划线作私有方法的前缀。你或许也想照着苹果公司的办法只拿一个下划线作前缀，这样做可能会惹来大麻烦: 如果从苹果公司提供的某个类中继承了一个子类，那么你在子类里可能会无意间覆写了父类的同名方法。鉴于此，苹果公司在文档中说，开发者不应该单用一个下划线作前缀。

### 总结

 - 1.给私有方法的名称加上前缀，这样可以很容易地将其同公共方法区分开。
 - 2.不要单用一个下划线做私有方法的前缀，因为这种做法是预留给苹果公司用的, 因为可能会复写了父类的方法。


## ㉑ 理解Objective-C 错误模型

首先要注意的是，`自动引用计数`(Automatic Reference Counting, ARC，参见第30条)在默认情况下不是`异常安全的`(exception safe)。具体来说，这意味着: 如果抛出异常，那么本应在作用域末尾释放的对象现在却不会自动释放了。如果想生成`异常安全`的代码，可以通过设置编译器的标志来实现。需要打开的编译器标志叫做`-fobjc-arc-exception`。

Objective-C语言现在所采用的办法是: 只在极其罕见的情况下抛出异常，异常抛出之后，无须考虑恢复问题，而且应用程序此时也应该退出。这就是说，不用再编写复杂的`异常安全`代码了。

`NSError`的用法更加灵活，因为经由此对象，我们可以把导致错误的原因回报给调用者。`NSError`对象里封装了三条信息:

- Error domain (错误范围，其类型为字符串)
- Error code (错误码，其类型为整数)
- User info (用户信息，其类型为字典)

在设计API时，`NSError`的第一种常见用法是通过委托协议来传递此错误。`NSError`的另外一种常见用法是:经由方法的`输出参数`返回给调用者。

### 总结

 - 1.只有发生了可使整个应用程序崩溃的严重错误时，才应使用异常。
 - 2.在错误不那么严重的情况下，可以指 `委托方法`(delegate method)来处理错误，也可以把错误信息放在`NSError`对象里，经由`输出参数`返回给调用者。



## ㉒ 理解NSCopying协议

因为以前开发程序时，会据此把内存分成不同的`区`(zone)，而对象会创建在某个区里面。现在不用了，每个程序只有一个区:`默认区`(default zone)。

若想使某个类支持拷贝功能，只需声明该类遵从`NSCopying协议`，并实现其中的那个方法即可。`copy`方法由NSObject实现，该方法只是以`默认区`为参数来调用`copyWithZone:`。我们总是想覆写`copy`方法，其实真正需要实现的却是`copyWithZone:`方法。

``` swift
// .h
@interface EOCPerson : NSObject <NSCopying>
@property (nonatomic, copy, readonly) NSString *firstName;
@property (nonatomic, copy, readonly) NSString *lastName;

- (id)initWithFirstName:(NSString*)firstName 
            andLastName:(NSString*)lastName;

@end

// .m
@implementation
- (id)copyWithZone:(NSZone*)zone {
    Person *copy = [[[self class] allocWithZone:zone] 
                    initWithFirstName:_firstName 
                          andLastName:_lastName];
    return copy;
}

@end
```

自定义`copy`方法需要遵循`NSCopying协议`并实现, 而自定义`mutableCopy`方法需要遵循`NSMutableCopying`协议并实现。

``` swift
- (id)mutableCopyWithZone:(NSZonen *)zone
```

### 有关容器类深拷贝

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-3/20171105145828579.png" width="70%" img/>
</center>

在容器类中, 如arrary, dictionary, set, 都有一个与深拷贝有关的实例方法, 下面以NSArray为例:

``` swift
- (id)initWithArarry: (NSArray *)array copyItems:(BOOL)copyItems
```

若copyItem参数设为YES，则该方法会向数组中的每个元素发送`copy`消息，用拷贝好的元素创建新的set，并将其返回给调用者。

但是, 因为没有专门定义深拷贝的协议，所以其具体执行方式由每个类来确定。如果需要在某对象上执行深拷贝，那么除非该类的文档说它是用深拷贝来实现`NSCopying`协议的，否则，要么寻找能够执行深拷贝的相关方法，要么自己编写方法来做。

### 总结

 - 1.若想令自己所写的对象具有拷贝功能，则需实现`NSCopying`协议
 - 2.如果自定义的对象分为可变版本和不可变版本，那么就要同时实现`NSCopying`与`NSMutableCopying`协议
 - 3.复制对象时需决定采用浅拷贝还是深拷贝，一般情况下应该尽量执行浅拷贝
 - 4.如果你写的对象需要深拷贝，那么可考虑新增一个专门执行深拷贝的方法



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


