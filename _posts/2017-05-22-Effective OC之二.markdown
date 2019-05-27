---
title:  "高效 OC开发之对象、消息、运行时"
date:   2017-05-22 01:38:32
categories: [iOS, Effective OC]
tags: [iOS, Effective OC]
---

属性的高效使用，消息转发机制，以及 一些runtime的介绍，如 Method Swizzle, objc_class等。

[![Contact](https://img.shields.io/badge/contact-wangyanchang21-green.svg)](https://github.com/wangyanchang21)  

------

- [⑥ 理解属性的概念](#-理解属性的概念)
- [⑦ 在对象内部尽量直接访问实例变量](#-在对象内部尽量直接访问实例变量)
- [⑧ 理解"对象等同性"](#-理解对象等同性)
- [⑨ 以"类簇模式"隐藏实现细节](#-以类簇模式隐藏实现细节)
- [⑩ 在既有类中使用关联对象存放自定义数据](#-在既有类中使用关联对象存放自定义数据)
- [⑪ 理解objc_msgSend的作用](#-理解objc_msgsend的作用)
- [⑫ 理解消息转发机制](#-理解消息转发机制)
- [⑬ 用“方法调配技术”调试“黑盒方法”](#-用方法调配技术调试黑盒方法)
- [⑭ 理解“类对象”的用意](#-理解类对象的用意)
- [相关资料](#相关资料)

-------


## ⑥ 理解属性的概念

想必你曾经也这样为某个类添加成员变量:

``` swift 
@interface Person : NSObject {
@public
    NSString *_firstName;
    NSString *_lastName;
@private
    NSString *_someInternalData;
}
```

但是有了属性之后, 也很少这么做了。 这种写法的问题在于, 对象的布局在编译器就已经固定了。 如果碰到成员变量`_firstName`编译器就会把其转化为`偏移量`(offset), 这个偏移量是`硬编码`(hardcode), 表示该变量距离存放对象的内存区域的起始地址有多远。这样是没问题的, 但是如果在中途有添加进来一个新的成员变量而且没有重新编译, 那这样对应成员变量的偏移量就不准确了。

``` swift
@interface Person : NSObject {
@public
    NSDate *_dateOfBirth;
    NSString *_firstName;
    NSString *_lastName;
@private
    NSString *_someInternalData;
}
```

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-2/20170511194512486.png" width="70%" img/>
</center>

OC对应的方法是, 把成员变量当做一种存储偏移量所用的特殊变量, 交由类对象保管。偏移量会在运行期查找, 如果类的定义变了, 那么存储的偏移量也就变了, 这样的话, 无论何时访问成员变量总能找到正确的偏移量。

#### 注:利用`->`存取成员变量的方式

上面代码中在`.h`文件中添加一些成员变量, 在外部使用的时候可以通过`实例->成员变量`来存取。`->`访问限制是, 在外部只能访问到`.h`中添加的`public`的成员变量, 在`.h`中添加的`private`的和`.m`中添加的成员变量只能在其类内部即`.m`中访问到。

### 属性的关键词

`@property`: 属性的声明关键词  
`@synthesize`: 意思是，由编译器自动添加默认的成员变量并生成相应的存取代码，以满足属性声明。  
`@dynamic`: 的意思是告诉编译器, 该属性的成员变量和存取方法由用户自己实现, 不自动生成。  
`@synthesize`和`@dynamic`是一对相反的关键字, 而在不做任何声明情况下, 默认为`@synthesize`的方式进行处理。  


`@synthesize`的作用:   

 - 1.添加属性对应的成员变量  
 - 2.规定了该属性声明的 `setter`方法,`getter`方法所操作的成员变量   
 - 3.如果@synthesize 省略不写,则自动生成对应属性的 `setter`方法，`getter`方法,默认操作的成员变量是’_’+属性名   
 - 4.检测手动实现了`@synthesize`, 就会按照你的要求生成成员变量名称并生成对应的setter getter 方法, 如`@synthesize` `name` =  `_myName`; 这样成员变量就是`_myName`了  
 - 5.在以下几种情况不会自动合成`setter`方法，`getter`方法, 而且也不会检测成员变量是否存在,也就不会帮助我们生成对应的成员变量,则需要我们自己添加成员变量: 1>同时重写了 `setter`方法，`getter`方法时; 2>重写了只读属性的 `getter`方法时; 3>使用了 `@dynamic` 时; 4>在 `@protocol` 中定义的所有属性; 5>在 `category` 中定义的所有属性; 6>重写的属性, 当你在子类中重写了父类中的属性，你必须使用 `@synthesize` 来手动合成`ivar`。  

### 内存管理语义

`assign`：`setter方法`只会执行对`纯量类型`(scalar type，例如CGFloat或NSInterger)的简单赋值操作

`strong`：此特质表明了属性定义了一种`拥有关系`。为这种属性设置新值时，会先保留新值，并释放旧值，然后再将新值设置上去。

`weak`：此特质表明了属性定义了一种`非拥有关系`。为这种属性设置新值时，`setter`方法既不保留新值，也不释放旧值。此特质类似`assign`，然而在属性所指的对象被销毁时，属性值也会清空。

`copy`：此特质所表达的所属关系与`strong`类似，然而`setter`方法并不保留新值，而是将其`copy`。当属性类型为`NSString *`时，经常用此特质来保护其封装性，保护数据不会在对象不知情的情况下被修改。

`unsafe_unretained`：此特质的语义和`assign`相同，但是它适用于`对象类型`，该特质表达一种`非拥有关系`，当目标对象被销毁时，属性值不会自动清空(“不安全”，unsafe)，这一点与`weak`不同。

### 原子特性

在默认情况下,由编译器所合成的方法会通过锁定机制确保其`原子性`(atomicity)。如果属性具备`nonatomic`特质，则不使用同步锁, 而默认是具有原子特性的, 若是自己定义存取方法，那么就应该遵从与属性特质相符的原子性。

具备`atomic`特质的获取方法会通过锁定机制来确保其操作的原子性。这也就是说，如果两个线程读写同一属性，那么不论何时，总能看到有效的属性值。若是不加锁的话(或者说使用`nonatomic`语义)，那么当其中一个线程正在改写某属性值时，另外一个线程也许会突然闯入，把尚未修改好的属性值读取出来。发生这种情况时，线程读到的属性值可能不对。

在iOS中使用同步锁的开销较大，这会带来性能问题。一般情况下并不要求属性必须是`原子的`，因为这并不能保证`线程安全`(thread safety)，若要实现`线程安全`的操作，还需采用更为深层的锁定机制才行。因此，开发iOS程序时一般都会使用`nonatomic`属性。

我们可以用下面的仿源码来理解`atomic`原理:

``` swift
- (NSString *)name {
    @synchronized(self) {
    return _name;
    }
}
- (void)setName:(NSString *)name {
    @synchronized(self) {
        _name = name;
    }
}
```

根据上面的代码, 可以看出 `atomic`只能保证`setter`, `getter`方法的线程安全, 并不能保证真正意义上的线程安全。比如说, 定义一个以`atomic`修饰的可变数组, 数组的`add`, `remove`和数组存储的内容的改变, 这些并不会通过数组的`setter`, `getter`方法, 所以并不会保证线程安全。

而且, 即使属性类型是上例中的`NSString`，一个线程在连续多次读取其属性值的过程中有别的线程在同时改写该值，那么即便将属性声明为`atomic`，也还是会读到不同的属性值。

想了解更多属性的内容, 可以参考我之前的博客: [属性详解(@property/@dynamic/@synthesize)](http://blog.csdn.net/wangyanchang21/article/details/50608097)


## ⑦ 在对象内部尽量直接访问实例变量

我在以前写过[property之 self.xx与_xx的区别](http://blog.csdn.net/wangyanchang21/article/details/50607651), 但是还是不近详细, 今天结合这本书所学的, 将这个问题彻底分析一下。 

### 在对象内部访问实例变量时, 通过属性访问与直接访问有什么区别? 

1.由于不经过Objective-C的`方法派发`(method dispatch)步骤，所以直接访问实例变量的速度当然比较快。在这种情况下，编译器所生成的代码会直接访问对象实例变量的那块内存。  
2.直接访问实例变量时，不会调用其`setter方法`，这就绕过了为相关属性所定义的`内存管理语义`。比方说，如果在ARC下直接访问一个声明为`copy`的属性，那么并不会拷贝该属性，只会保留新值并释放旧值。  
3.如果直接访问实例变量，那么不会触发`键值观测`(KVO)通知。这样做是否会产生问题，还取决于具体的对象行为。  
4.通过属性来访问有助于排查与之相关的错误，因为可以给`getter方法`和或`setter方法`中新增"断点"(breakpoint)，监控该属性的调用者及其访问时机。  

总之, 在写入实例变量时，通过其`setter方法`来做，而在读取实例变量时，则直接访问之。此办法既能提高读取操作的速度，又能控制对属性的写入操作。之所以要通过`setter方法`来写入实例变量，其首要原因在于，这样做能够确保相关属性的"内存管理语义"得以贯彻。但是，选用这种做法时，需注意几个问题。

### 在初始化方法中应该直接访问实例变量

这种情况下总是应该直接访问实例变量，因为子类可能会`重写`(override)`setter`方法。假设`SuperDog`有一个子类叫做`SubDog`，并重写了父类的某个属性的`setter`方法。当父类`SuperDog`的默认初始化方法中，可能会通过`setter`方法设置该属性。此时将会调用的将会是子类的`setter`方法。所以, 这里一定要直接访问实例变量来进行设置。举个例子说明为什么最好不要这样写。

首先新建一个`SuperDog`的类:

``` swift
@interface SuperDog : NSObject
@property (nonatomic, copy) NSString *name;
@end

@implementation SuperDog

- (instancetype)init {
    self = [super init];
    if (self) {
        self.name = @"";
        NSLog(@"类和方法:%s, 行数:%d，类型：%@", __PRETTY_FUNCTION__, __LINE__, NSStringFromClass([self class]));
    }
    return self;
}

- (void)setName:(NSString *)name {
    NSLog(@"类和方法:%s, 行数:%d，类型：%@", __PRETTY_FUNCTION__, __LINE__, @"不会执行到此方法");
    _name = @"SUPER";
}
@end
```

然后, 再新建一个类`SubDog`, 继承于`SuperDog`:

``` swift
@interface SubDog : SuperDog
@end

@implementation SubDog

@synthesize name = _name;

- (instancetype)init {
    self = [super init];
    if (self) {
        NSLog(@"类和方法:%s, 行数:%d，类型：%@", __PRETTY_FUNCTION__, __LINE__, NSStringFromClass([self class]));
        NSLog(@"类和方法:%s, 行数:%d，类型：%@", __PRETTY_FUNCTION__, __LINE__, NSStringFromClass([super class]));
    }
    return self;
}

- (void)setName:(NSString *)name {
    _name = @"SUB";
    NSLog(@"类和方法:%s, 行数:%d，类型：%@", __PRETTY_FUNCTION__, __LINE__, @"竟然会执行此方法!!!");
}
@end
```

当我们使用`SubDog`创建实例对象的时候, 

``` swift
    [[SubDog alloc] init];
```

打印结果如下:

``` swift
类和方法:-[SubDog setName:], 行数:26，类型：竟然会执行此方法!!!
类和方法:-[SuperDog init], 行数:17，类型：SubDog
类和方法:-[SubDog init], 行数:18，类型：SubDog
类和方法:-[SubDog init], 行数:19，类型：SubDog
```

所以说, 当我们在父类的`init`方法中, 使用了`self.xx`的方式, 而且其子类又重写了此属性的`setter`方法。 这时代码是这样执行的, `SubDog`在`init`时, 以`super` 的方式调用了`init`, 在`SuperDog`的`init`中, [self calss]应该是`SubDog`, 所以`self.xx`执行的是SubDog的setter方法。

但是在某些情况下必须要在初始化方方中调用`setter`方法: 如果待初始化的实例变量声明在超类中，而我们又无法在子类中直接访问此实例变量的话，那么就需要调用`setter方法`了。


### 总结

 - 1.在`对象内部`读取数据时，应该直接通过`实例变量`来读，而写入数据时，则应通过属性来写。
 - 2.在`初始化方法`及`dealloc方法`中，总是应该直接通过`实例变量`来读写数据。
 - 3.有时会使用`懒加载`(lazy initialization)配置某份数据，这种情况下，需要通过属性来读取数据。


## ⑧ 理解"对象等同性"

``` swift
NSString *foo = @"Badger 123";  
NSString *bar = [NSStringstringWithFormat:@"Badger %i", 123];  
BOOL equalA = (foo == bar); // equalAequalA = NO 
BOOL equalB = [foo isEqual:bar]; // equalBequalB = YES 
BOOL equalC = [foo isEqualToString:bar]; // equalCequalC = YES 
```

若使用 == 来判断, 但是必须是两个指针相同时, 才会返回YES。  
`isEqual:` 或者`isEqualToString:` 是当其内存地址一致时, 才会返回YES。所以, `isEqual:`是不可以判断父类对象与子类对象是否相同的。

另外, `isEqualToString:`比`isEqual:`方法快，后者还要执行额外的步骤，因为它不知道受测对象的类型。其他类型还有, `isEqualToArray:`, `isEqualToDictionary:`等方法。

### NSObject协议中判断等同性的关键方法

NSObject协议中有两个用于判断等同性的关键方法：

``` swift
- (BOOL)isEqual:(id)object;  
- (NSUInteger)hash; 
```

如果`isEqual:`方法判定两个对象相等，那么其`hash`方法也必须返回同一个值。但是，如果两个对象的`hash`方法返回同一个值，那么`isEqual:`方法未必会认为两者相等。

比如有下面这个类：

``` swift
@interface EOCPerson : NSObject  
@property (nonatomic, copy) NSString *firstName;  
@property (nonatomic, copy) NSString *lastName;  
@property (nonatomic, assign) NSUInteger age;  
@end 
```

我们认为，如果两个EOCPerson的所有字段均相等，那么这两个对象就相等。于是实现协议`isEqual:`方法可以写成：

``` swift
- (BOOL)isEqual:(id)object {  
    if (self == object) return YES;  
    if ([self class] != [object class]) return NO;  
 
    EOCPerson *otherPerson = (EOCPerson*)object;  
    if (![_firstName isEqualToString:otherPerson.firstName])  
        return NO;  
    if (![_lastName isEqualToString:otherPerson.lastName])  
        return NO;  
    if (_age != otherPerson.age)  
        return NO;  
    return YES;  
} 
```

接下来实现另一个`hash`方法。回想一下，根据等同性约定：若两对象相等，则其哈希码(hash)也相等，但是两个哈希码相同的对象却未必相等。这是能否正确重写`isEqual:`方法的关键所在。

``` swift
- (NSUInteger)hash {  
    NSUInteger firstNameHash = [_firstName hash];  
    NSUInteger lastNameHash = [_lastName hash];  
    NSUInteger ageHash = _age;  
    return firstNameHash ^ lastNameHash ^ ageHash;  
} 
```

这种做法既能保持较高效率，又能使生成的哈希码至少位于一定范围之内，而不会过于频繁地重复。编写`hash`方法时，应该用当前的对象做做实验，以便在减少碰撞频度与降低运算复杂程度之间取舍。

### 自定义高效率对比方法

如果经常需要判断等同性，那么可能会自己来创建等同性判定方法，因为无须检测参数类型，所以能大大提升检测速度。

在编写判定方法时，也应一并重写`isEqual:`方法。后者的常见实现方式为：如果受测的参数与接收该消息的对象都属于同一个类，那么就调用自已编写的判定方法，否则就交由超类来判断。例如，在EOCPerson类中可以实现如下两个方法：

``` swift
- (BOOL)isEqualToPerson:(EOCPerson*)otherPerson {  
    if (self == object) return YES;  
 
    if (![_firstName isEqualToString:otherPerson.firstName])  
        return NO;  
    if (![_lastName isEqualToString:otherPerson.lastName])  
        return NO;  
    if (_age != otherPerson.age)  
        return NO;  
    return YES;  
}  
 
- (BOOL)isEqual:(id)object {  
    if ([self class] == [object class]) {  
        return [self isEqualToPerson:(EOCPerson*)object];  
    } else {  
        return [super isEqual:object];  
    }  
} 
```

### 容器中可变类的等同性

把某个对象放入`collection`之后，就不应再改变其哈希码了。所以需要确保哈希码不是根据对象的`可变部分`(mutable portion)计算出来的，或是保证放入`collection`之后就不再改变对象内容了。

用一个`NSMutableSet`与几个`NSMutableArray`对象测试一下，就能发现这个问题了。首先把两个数组加入set中：

``` swift
NSMutableSet *set = [NSMutableSet new];  
 
NSMutableArray *arrayA = [@[@1, @2] mutableCopy];  
[set addObject:arrayA];  
NSMutableArray *arrayC = [@[@1] mutableCopy];  
[set addObject:arrayC];  
NSLog(@"set = %@", set);  
// Output: set = {((1),(1,2))} 
```

由于arrayC与set里已有的对象不相等，所以现在set里有两个数组了：其中一个是最早加入的，另一个是刚才新添加的。最后，我们改变arrayC的内容，令其和最早加入set的那个数组相等：

``` swift
[arrayC addObject:@2];  
NSLog(@"set = %@", set);  
// Output: set = {((1,2),(1,2))} 
```

set中居然可以包含两个彼此相等的数组！根据set的语义是不允许出现这种情况的，然而现在却无法保证这一点了，因为我们修改了set中已有的对象。

这个例子足以说明, 把某对象放入`collection`之后改变其内容将会造成何种后果。笔者并不是说绝对不能这么做，而是说如果真要这么做，那就得注意其隐患，并用相应的代码处理可能发生的问题。

### 总结

 - 1.若想检测对象的等同性，请提供`isEqual:`与`hash`方法。  
 - 2.相同的对象必须具有相同的哈希码，但是两个哈希码相同的对象却未必相同。   
 - 3.不要盲目地逐个检测每条属性，而是应该依照具体需求来制定检测方案。   
 - 4.编写`hash`方法时，应该使用计算速度快而且哈希码碰撞几率低的算法。   
 - 5.把某个对象放入`collection`之后，就不应再改变其哈希码了, 所以尽量使用不可变对象。   


## ⑨ 以"类簇模式"隐藏实现细节

Objective-C的系统框架中普遍使用此模式。比如，iOS的用户界面框架(user interface framework)`UIKit`中就有一个名为`UIButton`的类。想创建按钮，需要调用下面这个`类方法`(class method)：
 
``` swift
+ (UIButton*)buttonWithType:(UIButtonType)type; 
```

该方法所返回的对象，其类型取决于传入的按钮类型(button type)。然而，不管返回什么类型的对象，它们都继承自同一个基类：UIButton。这么做的意义在于：UIButton类的使用者无须关心创建出来的按钮具体属于哪个子类，也不用考虑按钮的绘制方式等实现细节。使用者只需明白如何创建按钮，如何设置像标题(title)这样的属性，如何增加触摸动作的目标对象等问题就好。


### Cocoa里的类簇

系统框架中有许多`类簇`。大部分`collection`类都是类簇，例如NSArray与NSMutableArray, NSDictionary和NSMutableDictionary, 以及NSNumber。

| 类 | 真身 |
| :-----: | :-----: |
| NSArray | __NSArrayI |
| NSMutableArray | __NSArrayM |
| NSDictionary | __NSDictionaryI |
| NSMutableDictionary |	__NSDictionaryM |


``` swift
if ([maybeAnArray class] == [NSArray class]) {  
        // Will never be hit  
} 
```

[maybeAnArray class]所返回的类绝不可能是`NSArray`类本身，因为由NSArray的初始化方法所返回的那个实例其类型是隐藏在类簇公共接口(public facade)后面的某个内部类型(internal type)。

所以, 若想判断某对象是否位于类簇中，不要直接检测两个`类对象`是否等同，而应该采用下列代码:

``` swift
if ([maybeAnArray isKindOfClass:[NSArray class]]) {  
        // Will be hit  
} 
```

### 向Cocoa的类簇中新增实体子类

我们经常需要向类簇中新增实体子类，所需遵循的规范一般都会定义于基类的文档之中，而且需要遵守几条规则:   
1.子类应该继承自类簇中的抽象基类。若要编写`NSArray类簇`的子类，则需令其继承自不可变数组的基类或可变数组的基类。   
2.子类应该定义自己的数据存储方式。子类必须用一个实例变量来存放数组中的对象。NSArray本身只不过是包在其他隐藏对象外面的壳，它仅仅定义了所有数组都需具备的一些接口。   
3.子类应当重写超类文档中指明需要重写的方法。   

### 总结

 - 1.类簇模式可以把实现细节隐藏在一套简单的公共接口后面。   
 - 2.系统框架中经常使用类簇。   
 - 3.从类簇的公共抽象基类中继承子类时要当心，若有开发文档，则应首先阅读。   


## ⑩ 在既有类中使用关联对象存放自定义数据

### AssociatedObject

有时需要在对象中存放相关信息, 而且可以给某对象关联许多其他对象，这些对象通过`键`来区分。`runtime`中设置关联的方法如下:

``` swift
// 关联
 void objc_setAssociatedObject(id object, void*key, id value, objc_AssociationPolicy policy)
// 获取
id objc_getAssociatedObject(id object, void*key)
// 移除
void objc_removeAssociatedObjects(id object)
```

存储对象值的时候，可以指明`存储策略`(storage policy)，用以维护相应的`内存管理语义`。存储策略由名为objc_AssociationPolicy的枚举所定义。假如关联对象成为了属性，那么它就会具备对应的语义。

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-2/20170520140827285.png" width="70%" img/>
</center>

设置关联对象时用的键(key)是个`不透明的指针`(opaque pointer)。如果在两个键上调用`isEqual:`方法的返回值是YES，那么NSDictionary就认为二者相等；然而在设置关联对象值时，若想令两个键匹配到同一个值，则二者必须是完全相同的指针才行。鉴于此，在设置关联对象值时，通常使用静态全局变量做键。

### 总结

 - 1.可以通过`关联对象`机制来把两个对象连起来。
 - 2.定义关联对象时可指定内存管理语义，用以模仿定义属性时所采用的`拥有关系`与`非拥有关系`。
 - 3.只有在其他做法不可行时才应选用关联对象，因为这种做法通常会引入难于查找的bug。


## ⑪ 理解objc_msgSend的作用

在对象上调用方法是Objective-C中经常使用的功能。用Objective-C的术语来说，这叫做`传递消息`(pass a message)。

``` swift
id returnValue = [someObject messageName:parameter]; 
```

### 原理

在上例中，someObject叫做`接收者`(receiver)，messageName叫做`选择子`(selector)。选择子与参数合起来称为`消息`(message)。编译器看到此消息后，将其转换为一条标准的C语言函数调用，所调用的函数乃是消息传递机制中的核心函数，叫做`objc_msgSend`，其`原型`(prototype)如下：

``` swift
void objc_msgSend(id self, SEL cmd, ...) 
```

这是个`参数个数可变的函数`(variadic function)，能接受两个或两个以上的参数。第一个参数代表接收者，第二个参数代表选择子(SEL是选择子的类型)，后续参数就是消息中的那些参数，其顺序不变。选择子指的就是方法的名字。`选择子`与`方法`这两个词经常交替使用。编译器会把刚才那个例子中的消息转换为如下函数：

``` swift
id returnValue = objc_msgSend(someObject, @selector(messageName:), parameter); 
```

`objc_msgSend`函数会依据接收者与选择子的类型来调用适当的方法。为了完成此操作，该方法需要在接收者所属的类中搜寻其`方法列表`(list of methods)，如果能找到与选择子名称相符的方法，就跳至其实现代码。若是找不到，那就沿着继承体系继续向上查找，等找到合适的方法之后再跳转。如果最终还是找不到相符的方法，那就执行`消息转发`(message forwarding)操作。

这么说来，想调用一个方法似乎需要很多步骤。所幸`objc_msgSend`会将匹配结果缓存在`快速映射表`(fast map)里面，每个类都有这样一块缓存，若是稍后还向该类发送与选择子相同的消息，那么执行起来就很快了。当然啦，这种`快速执行路径`(fast path)还是不如`静态绑定的函数调用操作`(statically bound function call)那样迅速，不过只要把选择子缓存起来了，也就不会慢很多。

`objc_msgSend`等函数一旦找到应该调用的方法实现之后，就会`跳转过去`。之所以能这样做，是因为Objective-C对象的每个方法都可以视为简单的C函数，其原型如下：

``` swift
<return_type> Class_selector(id self, SEL _cmd, ...) 
```

真正的函数名和上面写的可能不太一样，笔者用`类`(class)和`选择子`(selector)来命名是想解释其工作原理。每个类里都有一张表格，其中的指针都会指向这种函数，而选择子的名称则是查表时所用的`键`。`objc_msgSend`等函数正是通过这张表格来寻找应该执行的方法并跳至其实现的。

如果某函数的最后一项操作是调用另外一个函数，那么就可以运用`尾调用优化`技术。编译器会生成调转至另一函数所需的指令码，而且不会向调用堆栈中推入新的`栈帧`(frame stack)。只有当某函数的最后一个操作仅仅是调用其他函数而不会将其返回值另作他用时，才能执行`尾调用优化`。这项优化对`objc_msgSend`非常关键，如果不这么做的话，那么每次调用Objective-C方法之前，都需要为调用`objc_msgSend`函数准备`栈帧`。


### 其他函数

`objc_msgSend_stret`

如果待发送的消息要返回结构体，那么可交由此函数处理。只有当CPU的寄存器能够容纳得下消息返回类型时，这个函数才能处理此消息。若是返回值无法容纳于CPU寄存器中(比如说返回的结构体太大了)，那么就由另一个函数执行派发。此时，那个函数会通过分配在栈上的某个变量来处理消息所返回的结构体。

`objc_msgSend_fpret`

如果消息返回的是浮点数，那么可交由此函数处理。在某些架构的CPU中调用函数时，需要对“浮点数寄存器”(floating-point register)做特殊处理，也就是说，通常所用的objc_msgSend在这种情况下并不合适。这个函数是为了处理x86等架构CPU中某些令人稍觉惊讶的奇怪状况。

`objc_msgSendSuper`

如果要给超类发消息，例如[super message:parameter]，那么就交由此函数处理。也有另外两个与objc_msgSend_stret和objc_msgSend_fpret等效的函数，用于处理发给super的相应消息。

### 总结

 - 1.消息由接收者、选择子及参数构成。给某对象`发送消息`(invoke a message)也就相当于在该对象上`调用方法`(call a method)。  
 - 2.发给某对象的全部消息都要由`动态消息派发系统`(dynamic message dispatch system)来处理，该系统会查出对应的方法，并执行其代码。  


## ⑫ 理解消息转发机制

在编译期向类发送了其无法解读的消息并不会报错，因为在运行期可以继续向类中添加方法。当对象接收到无法解读的消息后，就会启动`消息转发`(message forwarding)机制，程序员可经由此过程告诉对象应该如何处理未知消息。

### 举例

```
-[__NSCFNumber lowercaseString]: unrecognized selector sent to instance 0x87  
```

上面这段异常信息是由NSObject的“`doesNotRecognizeSelector:`方法所抛出的，此异常表明：消息接收者的类型是_ _NSCFNumber，而该接收者无法理解名为lowercaseString的选择子。

### 原理

消息转发分为两大阶段。第一阶段先征询接收者，所属的类，看其是否能动态添加方法，以处理当前这个`未知的选择子`(unknown selector)，这叫做`动态方法解析`(dynamic method resolution)。第二阶段涉及`完整的消息转发机制`(full forwarding mechanism)。如果运行期系统已经把第一阶段执行完了，那么接收者自己就无法再以动态新增方法的手段来响应包含该选择子的消息了。

此时，运行期系统会请求接收者以其他手段来处理与消息相关的方法调用。这又细分为两小步。首先，请接收者看看有没有其他对象能处理这条消息。若有，则运行期系统会把消息转给那个对象，于是消息转发过程结束，一切如常。若没有`备援的接收者`(replacement receiver)，则启动完整的消息转发机制，运行期系统会把与消息有关的全部细节都封装到`NSInvocation对象`中，再给接收者最后一次机会，令其设法解决当前还未处理的这条消息。

### 动态方法解析

对象在收到无法解读的消息后，首先将调用其所属类的下列类方法：

``` swift
// 实例的selector
+ (BOOL)resolveInstanceMethod:(SEL)selector;
// 类的selector
+ (BOOL)resolveClassMethod:(SEL)selector; 
```

该方法的参数就是那个未知的选择子，其返回值为`Boolean`类型，表示这个类是否能新增一个实例方法用以处理此选择子。在继续往下执行转发机制之前，本类有机会新增一个处理此选择子的方法。

使用这种办法的前提是：相关方法的实现代码已经写好，只等着运行的时候动态插在类里面就可以了。下列代码演示了如何用`resolveInstanceMethod:`来实现`@dynamic`属性：
 
``` swift
id autoDictionaryGetter(id self, SEL _cmd);  
void autoDictionarySetter(id self, SEL _cmd, id value);  
 
+ (BOOL)resolveInstanceMethod:(SEL)selector {  
    NSString *selectorString = NSStringFromSelector(selector);  
    if ( /* selector is from a @dynamic property */ ) {  
        if ([selectorString hasPrefix:@"set"]) {  
            class_addMethod(self, selector, (IMP)autoDictionarySetter, "v@:@");  
        } else {  
            class_addMethod(self, selector, (IMP)autoDictionaryGetter, "@@:");  
        }  
        return YES;  
    }  
return [super resolveInstanceMethod:selector];  
} 
```

### 备援接收者

当前接收者还有第二次机会能处理未知的选择子，在这一步中，运行期系统会问它：能不能把这条消息转给其他接收者来处理。与该步骤对应的处理方法如下：

``` swift
- (id)forwardingTargetForSelector:(SEL)selector;
```

方法参数代表未知的选择子，若当前接收者能找到备援对象，则将其返回，若找不到，就返回nil。在一个对象内部，可能还有一系列其他对象，该对象可经由此方法将能够处理某选择子的相关内部对象返回。  

请注意，我们无法操作经由这一步所转发的消息。若是想在发送给备援接收者之前先修改消息内容，那就得通过完整的消息转发机制来做了。

### 完整的消息转发

如果转发算法已经来到这一步的话，那么唯一能做的就是启用完整的消息转发机制了。首先创建`NSInvocation`对象，把与尚未处理的那条消息有关的全部细节都封于其中。此对象包含选择子、目标(target)及参数。在触发`NSInvocation`对象时，`消息派发系统`(message-dispatch system)将亲自出马，把消息指派给目标对象。

此步骤会调用下列方法来转发消息：

``` swift
- (void)forwardInvocation:(NSInvocation*)invocation 
```

这个方法可以实现得很简单：只需改变调用目标，使消息在新目标上得以调用即可。实现此方法时，若发现某调用操作不应由本类处理，则需调用超类的同名方法。这样的话，继承体系中的每个类都有机会处理此调用请求，直至`NSObject`。如果最后调用了NSObject类的方法，那么该方法还会继而调用`doesNotRecognizeSelector:`以抛出异常，此异常表明选择子最终未能得到处理。

### 消息转发全流程

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-2/20170522001844727.png" width="80%" img/>
</center>

接收者在每一步中均有机会处理消息。步骤越往后，处理消息的代价就越大。最好能在第一步就处理完，这样的话，运行期系统就可以将此方法缓存起来了。如果这个类的实例稍后还收到同名选择子，那么根本无须启动消息转发流程。若想在第三步里把消息转给备援的接收者，那还不如把转发操作提前到第二步。因为第三步只是修改了调用目标，这项改动放在第二步执行会更为简单，不然的话，还得创建并处理完整的`NSInvocation`。


### 总结

 - 1.若对象无法响应某个选择子，则进入消息转发流程。  
 - 2.通过运行期的动态方法解析功能，我们可以在需要用到某个方法时再将其加入类中。  
 - 3.对象可以把其无法解读的某些选择子转交给其他对象来处理。  
 - 4.经过上述两步之后，如果还是没办法处理选择子，那就启动完整的消息转发机制。  

## ⑬ 用“方法调配技术”调试“黑盒方法”

与给定的选择子名称相对应的方法也可以在运行期改变，此方案经常称为`方法调配`(method swizzling)。而且我们既不需要源代码，也不需要通过继承子类来覆写方法就能改变这个类本身的功能。这样一来，新功能将在本类的所有实例中生效，而不是仅限于覆写了相关方法的那些子类实例。

类的方法列表会把选择子的名称映射到相关的方法实现之上，使得`动态消息派发系统`能够据此找到应该调用的方法。这些方法均以函数指针的形式来表示，这种指针叫做`IMP`，其原型如下：

``` swift
id (*IMP)(id, SEL, ...) 
```

### 方法介绍

举个例子, 正常情况下NSString的一些方法: 
<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-2/20170522004101146.png" width="80%" img/>
</center>

Objective-C运行期系统提供的几个方法都能够用来操作这张表。开发者可以向其中新增选择子，也可以改变某选择子所对应的方法实现，还可以交换两个选择子所映射到的指针。经过几次操作之后，类的方法表就会变成下图这个样子。

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-2/20170522004111053.png" width="80%" img/>
</center>


想交换方法实现，可用下列函数：

``` swift
void method_exchangeImplementations(Method m1, Method m2) 
``` 

此函数的两个参数表示待交换的两个方法实现，而方法实现则可通过下列函数获得：

``` swift
Method class_getInstanceMethod(Class aClass, SEL aSelector) 
```

### Method Swizzling应用

`Method Swizzling`在我的博客中也曾探究过: [Runtime之黑魔法-Method Swizzling]。(http://blog.csdn.net/wangyanchang21/article/details/61199865)

### 总结

 - 1.在运行期，可以向类中新增或替换选择子所对应的方法实现。
 - 2.使用另一份实现来替换原有的方法实现，这道工序叫做`方法调配`，开发者常用此技术向原有实现中添加新功能。
 - 3.此做法只在调试程序时有用, 而且很少有人在调试程序之外的场合用上述`方法调配技术`来永久改动某个类的功能。
 - 4.一般来说，只有调试程序的时候才需要在运行期修改方法实现，这种做法不宜滥用。


## ⑭ 理解“类对象”的用意


### objc_class

``` swift
typedef struct objc_class *Class;  
struct objc_class {
  Class isa; //isa指针指向Meta Class，因为Objc的类的本身也是一个Object，为了处理这个关系，runtime就创造了Meta Class，当给类发送[NSObject alloc]这样消息时，实际上是把这个消息发给了Class Object
  #if !__OBJC2__
  Class super_class; // 父类
  const char *name; // 类名
  long version; // 类的版本信息，默认为0
  long info; // 类信息，供运行期使用的一些位标识
  long instance_size; // 该类的实例变量大小
  struct objc_ivar_list *ivars; // 该类的成员变量链表
  struct objc_method_list **methodLists; // 方法定义的链表
  struct objc_cache *cache; // 方法缓存，对象接到一个消息会根据isa指针查找消息对象，这时会在method Lists中遍历，如果cache了，常用的方法调用时就能够提高调用的效率。
  struct objc_protocol_list *protocols; // 协议链表
  #endif
  } OBJC2_UNAVAILABLE;
```

objc在向一个对象发送消息时，`runtime`库会根据对象的isa指针找到该对象实际所属的类，然后在该类中的方法列表以及其父类方法列表中寻找方法运行，然后在发送消息的时候，`objc_msgSend`方法不会返回值，所谓的返回内容都是具体调用时执行的。 那么，如果向一个nil对象发送消息，首先在寻找对象的`isa指针`时就是0地址返回了，所以不会出现任何错误。

所有父类的成员变量和自己的成员变量都会存放在该对象所对应的存储空间中.
每一个类对象内部都有一个`isa指针`,指向他的类对象, 类对象中存放着本对象的对象方法列表(对象能够接收的消息列表，保存在它所对应的类对象中), 成员变量的列表, 属性列表。
它内部也有一个`isa指针`指向`元对象`(meta class),元对象内部存放的是类方法列表, 类对象内部还有一个`superclass指针`,指向他的父类对象。

### objc对象

每个 Objective-C 对象都有相同的结构，如下图所示：

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-2/20170522012117687.png" width="30%" img/>
</center>

根对象就是`NSObject`，它的`superclass`指针指向`nil`
类对象既然称为对象，那它也是一个实例。类对象中也有一个`isa指针`指向它的`元类`(meta class)，即类对象是元类的实例。元类内部存放的是类方法列表，根元类的`isa指针`指向自己，`superclass指针`指向NSObject类。

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-2/20170522012249277.png" width="80%" img/>
</center>


### 在类继承体系中查询类型信息

可以用类型信息查询方法来检视类继承体系。`isMemberOfClass:`能够判断出对象是否为某个特定类的实例，而`isKindOfClass:`则能够判断出对象是否为某类或其派生类的实例。例如：

``` swift
NSMutableDictionary *dict = [NSMutableDictionary new];  
[dict isMemberOfClass:[NSDictionary class]]; ///< NO 
[dict isMemberOfClass:[NSMutableDictionary class]]; ///< YES 
[dict isKindOfClass:[NSDictionary class]]; ///< YES 
[dict isKindOfClass:[NSArray class]]; ///< NO 
```

也可以用比较类对象是否等同的办法来做。若是如此，那就要使用==操作符。

``` swift
id object = /* ... */;  
if ([object class] == [EOCSomeClassclass]) {  
    // 'object' is an instance of EOCSomeClass  
} 
```

通常情况下，如果在此种代理对象上调用class方法，那么返回的是代理对象本身(此类是NSProxy的子类)，而非接受的代理的对象所属的类。然而，若是改用`isKindOfClass:`这样的类型信息查询方法，那么代理对象就会把这条消息转给`接受代理的对象`(proxied object)。也就是说，这条消息的返回值与直接在接受代理的对象上面查询其类型所得的结果相同。因此，这样查出来的类对象与通过`class方法`所返回的那个类对象不同，`class方法`所返回的类表示发起代理的对象，而非接受代理的对象。

### 总结

 - 1.每个实例都有一个指向Class对象的指针，用以表明其类型，而这些Class对象则构成了类的继承体系。
 - 2.如果对象类型无法在编译期确定，那么就应该使用类型信息查询方法来探知。
 - 3,尽量使用类型信息查询方法来确定对象类型，而不要直接比较类对象，因为某些对象可能实现了消息转发功能。


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


