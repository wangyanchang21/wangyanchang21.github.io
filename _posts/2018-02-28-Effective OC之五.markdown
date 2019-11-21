---
title:  "高效 OC开发之内存管理"
date:   2018-02-28 17:18:43
categories: [iOS, Effective OC, 内存管理]
tags: [iOS, Effective OC, 内存管理]
---

引用计数，ARC的工作原理，dealloc和循环引用的问题，以及其它关于内存管理的部分特性。 

[![Contact](https://img.shields.io/badge/contact-wangyanchang21-green.svg)](https://github.com/wangyanchang21)  

------

- [㉙ 理解引用计数](#-理解引用计数)
- [㉚ 以ARC简化引用计数](#-以arc简化引用计数)
- [㉛ 在dealloc方法中只释放引用并解除监听](#-在-dealloc方法中只释放引用并解除监听)
- [㉜ 编写“异常安全代码”时留意内存管理问题](#-编写异常安全代码时留意内存管理问题)
- [㉝ 以弱引用避免保留环](#-以弱引用避免保留环)
- [㉞ 以“自动释放池块”降低内存峰值](#-以自动释放池块降低内存峰值)
- [㉟ 用“僵尸对象”调试内存管理问题](#-用僵尸对象调试内存管理问题)
- [㊱ 不要使用 retain Count](#-不要使用-retain-count)
- [相关资料](#相关资料)

-------

## ㉙ 理解引用计数

Objective-C语言通过`引用计数`来管理内存。在iOS4及之前都是手动管理内存(Manual Reference Counting, `MRC`), 而从iOS5开始, 就支持自动引用计数(Automatic Reference Counting, `ARC`)了, 所以就变得更为简单了。`ARC`几乎把所有内存管理事宜都交由编译器来决定, 开发者只需专注于业务逻辑。

在`ARC`中, 所有与引用计数有关的方法都无法编译, 所以在开启`ARC`功能时不能直接调用的内存管理的方法。


### 引用计数工作原理

在引用计数架构下, 对象有个计数器,用以表示当前有多少个事物想令此对象继续存活下去。这在 Objective-C中叫做`引用计数`(Reference Counting)。

NSObject协议声明了下面三个方法用于操作计数器, 以递增或递减其值:  
`Retain`: 递增保留计数。  
`Release`: 递减保留计数。  
`Autorelease`: 待稍后清理`自动释放池`(autorelease pool)时, 再递减保留计数。

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-5/20180223183026217.png" width="50%" img/>
</center>


在MRC中, 一般如下使用: 

``` swift
NSMutablearray *array = [[NSMutableArray alloc] init];
NSNumber *number =[[NSNumber alloc] initWithInt: 1337];
[array addObject:number];
[number release];

∥ do something with array
[array release];
```

当对象的引用计数降至0,那么number对象所占内存也许会回收, 再向其发送消息时, 可能就将使程序崩溃了。这里只说`可能`, 而没说`一定`, 因为对象所占的内存在`解除分配` (deallocated)之后, 只是放回`可用内存池`(avaiable pool)。如果向其发送消息时, 尚未覆写对象内存, 那么该对象仍然有效, 这时程序不会崩溃。所以, 为避免在不经意间使用了无效对象, 一般调用完 release之后都会清空指针, 将对象置为`nil`。

### 属性存取方法中的内存管理

访问属性时,会用到相关实例变量的获取方法及设置方法。若属性为`strong关系`(strong relationship), 则设置的属性值会保留。比方说, 有个名叫foo的属性由名为_foo的实例变量所实现, 那么该属性的设置方法应该是这样:

``` swift
- (void)setFoo:(id)foo {
	[foo retain];
	[_foo release];
	_foo = foo;
}
```

### 自动释放池

在 Objective-C的引用计数架构中, 调用 `release`会立刻递减对象的保留计数(而且还有可能令系统回收此对象), 然而调用`autorelease`时, 会在稍后递减计数, 通常是在下一次`事件循环`(event loop)时递减, 不过也可能执行得更早些(参见第34条)。

`autorelease`方法, 它会在稍后释放对象, 从而给调用者留下了足够长的时间, 使其可以在需要时先保留返回值。实际上, 释放操作会在清空最外层的自动释放池(参见第34条)时执行, 除非你有自己的自动释放池, 否则这个时机指的就是当前线程的下一次事件循环。

### 保留环(循环引用)

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-5/20180224113513746.png" width="50%" img/>
</center>


呈环状相互引用的多个对象, 将导致内存泄漏, 因为循环中的对象其保留计数不会降为0。所以, 通常通过`弱引用`(weak reference,参见第33条), 或是从外界命令循环中的某个对象不再保留另外一个对象。从而打破保留环, 避免内存泄漏。


### 总结

1.引用计数机制通过可以递增递减的计数器来管理内存。对象创建好之后,其保留计数至少为1。若保留计数为正,则对象继续存活。当保留计数降为0时,对象就被销毁了
2.在对象生命期中,其余对象通过引用来保留或释放此对象。保留与释放操作分别会递增及递减保留计数。


## ㉚ 以ARC简化引用计数

在Clang编译器项目带有一个`静态分析器`(static analyzer)用于指名程序里引用计数出问题的地方。

在使用`ARC`时一定要记住, 引用计数实际上还是要执行的, 只不过保留与释放操作现在是由`ARC`自动为你添加。由于`ARC`会自动执行 `retain`、`release`、`autorelease`、`decalloc`等操作, 所以直接在`ARC`下调用这些内存管理方法是非法的。

实际上,  `ARC`在调用这些方法时, 并不通过普通的 Objective-C消息派发机制,而是直接调用其底层C语言版本。这样做性能更好, 因为保留及释放操作需要频繁执行, 所以直接调用底层函数能节省很多CPU周期。

### 使用ARC时必须遵循的方法命名规则

将内存管理语义在方法名中表示出来早已成为 Objective-C的惯例, 而`ARC`则将之确立为硬性规定。若方法名以下列词语开头, 则其返回的对象归调用者所有: `alloc`、`new`、`copy`、`mutable Copy`。若调用上述开头的方法就要负责释放返回的对象。也就是说, 这些对象在`MRC`中需要你手动的进行释放。若方法名不以上述四个词语开头, 返回的对象就不需要你手动去释放, 因为在方法内部将会自动执行一次`autorelease`方法。

``` swift
+ (EOCPerson)newPerson{  
EOCPerson *person = [[EOCPerson alloc]init];  
return person;  
/* 
这个方法以new开头，那么不需要retain、release和autorelease了。
*/  
}  

+ (EOCPerson)somePerson{  
EOCPerson *person = [[EOCPerson alloc]init];  
return person;  
/* 
这个方法以new、alloc等这些拥有`对象`的词语开头，在MRC中这里的代码相当于 return [person autorelease]。
*/  
}  

- (void)doSomething{  
EOCPerson *personOne = [EOCPerson newPerson];  
EOCPerson *personTwo = [EOCPerson somePerson];  
/* 
personOne和personTwo已经到了作用范围，因此ARC需要清理他们。
personOne 拥有对象，所以需要release, 在MRC中这里需要添加代码 [personOne release]。
personTwo 不拥有对象，所以不需要release。
*/  
}
```

而且`ARC`是包含运行期组件的, `ARC`中还做了其他的很多优化, 下面举个例子:

``` swift
EOCPerson *tmp= [EOCPerson personWithName: @"Bob Smith"];
_myPerson = [tmp retain];
```

此时应该能看出来, `personWithname:`方法里会自动添加的 `autorelease`与后面的 `retain`都是多余的。当然`ARC`可以在运行期检测到这一对多余的操作。为了优化代码, 在方法中返回自动释放的对象时, 要执行一个特殊函数。下面这段代码演示了`ARC`是如何通过这些特殊函数来优化程序的:

``` swift
// Within EOCperson class

+ (EOCPerson*)personWithName:(Nsstring *)name {
	EOCPerson *person =[[Eocperson alloc] init];
	person.name = name;
	return objc_autoreleaseReturnValue(person);
}

// Code using EOCPerson class
Eocperson *tmp = [EOCPerson personWithName: @"Matt Galloway"];
_myPerson = objc_retainAutoreleasedReturnValue(tmp);
```

此时不直接调用 `autorelease`方法, 而是改为调用`objc_autoreleaseReturnValue`。此函数会检测之后的代码, 会根据返回的对象是否执行 `retain`操作, 来设置全局数据结构中的一个标志位, 决定是否执行 `autorelease`操作。与之相似,  `retain`方法将改为执行 `objc_retainAutoreleasedReturnValue`, 此函数要检测刚才提到的那个标志位, 根据标志位来决定是否执行 `retain`操作。当然设置并检测标志位要比调用 `autorelease`和 `retain`更快。

更具体的描述可以参考另一篇博客: [ARC到底帮我们做了哪些工作?](https://wangyanchang21.github.io/2018/ARC%E5%88%B0%E5%BA%95%E5%B8%AE%E6%88%91%E4%BB%AC%E5%81%9A%E4%BA%86%E5%93%AA%E4%BA%9B%E5%B7%A5%E4%BD%9C/#%E4%B8%8D%E9%9C%80%E8%A6%81%E6%89%8B%E5%8A%A8%E9%87%8A%E6%94%BE%E8%BF%94%E5%9B%9E%E5%AF%B9%E8%B1%A1%E7%9A%84%E6%96%B9%E6%B3%95)

为了求得最佳效率, 这些特殊函数的实现代码都因处理器而异。下面这段伪代码大概描述了其中的实现原理:

``` swift
id objc_autoreleaseReturnValue(id object) {
    if ( //调用者将会执行retain ) {
          set_flag(object);
          return object;
    } else {
          return [object autorelease];    
    }
}
id objc_retainAutoreleasedReturnValue(id object) {
    if (get_flag(object))  {
        clear_flag(object);
        return object;
    } else {
        return [object retain];
    }
}
```

将内存管理交由编译器和运行期组件来做, 可以使代码得到多种优化, 上面所讲的只是其中一种。我们由此应该可以了解到`ARC`所带来的好处。


### 变量的内存管理语义

`ARC`也会处理局部变量与实例变量的内存管理。默认情况下,每个变量都是指向对象的强引用。比如下面这段代码:

``` swift
- (void)setup {
	_object = [EOCOtherclass new];
}
```

在手动管理引用计数时, 实例变量 _object并不会自动保留其值, 而在`ARC`环境下则会这样做。也就是说, 若在`ARC`下编译 setup方法, 则其代码会变为:

``` swift
- (void)setup {
	id temp = [EOCOtherclass new];
	_object = [tmp retain];
	[tmp release];
}
```

当然, 原理如此, 但实际上 `retain`和 `release`是可以优化消去的。所以, `ARC`会将这两个操作化简掉, 执行的代码还是和原来是一样的。
不过, 在编写`setter方法`时, 使用`ARC`会简单些。在`MRC`下, 你可能会写成下面这样:

``` swift
- (void)setObject:(id)object {
	[_object release];
	_object = [object retain];
}
```

但是这样写会出问题。假如 object在 `release`后的引用计数降为0, 从而导致系统将其回收, 接下来再执行 `retain`操作, 就会令应用程序崩溃。使用`ARC`之后, 就不可能发生这种疏失了。`ARC`自动的先保留新值, 再释放旧值, 最后设置实例变量, 使其安全的存储。

在应用程序中，可用下列修饰符来改变局部变量与实例变量的语义：  
`__strong`: 默认语义保留此值。
`__unsafe_unretained：不保留此值，可能不安全，因为再次使用变量时，其对象可能已经回收了。   
`__weak`：不保留此值，变量可以安全使用，系统把对象回收后会自动清空它。   
`__autorelease`：把对象按照引用传递给方法时，使用这个特殊的修饰符。此值在方法返回时自动释放。   


### ARC如何清理实例变量

在`MRC`下, 你肯定会这么去重写`dealloc`方法: 

``` swift
- (void)dealloc {
	[_foo release];
	[_someIvar release];
	_foo = nil;
	_someIvar = nil;    
    [super dealloc];
}
```

用了`ARC`之后, 就不需要再编写这种`dealloc`方法了, 因为`ARC`会借用 Objective-C++的一项特性来生成清理例程(cleanup routine)。回收 Objective-C++对象时, 待回收的对象会调用所有C++对象的析构函数(destructor)。编译器如果发现某个对象里含有C++对象, 就会生成名为 .cxx_destruct的方法。而`ARC`则借助此特性, 在该方法中生成清理内存所需的代码。

如果有非 Objective-C的对象, 比如 `Core Foundation`中的对象或是由 `malloc`分配在堆中的内存, 那么仍然需要清理。

``` swift
- (void)dealloc{  
	CFRelease(_coreFoundationObject);  
	free(_heapAllocateMemoryBlob);  
}  
```

### 重写内存管理方法

我们经常覆写 `release`方法, 将其替换为`空操作`(no-op)。但在`ARC`环境下不能这么做, 因为会干扰到`ARC`分析对象生命期的工作。


### 总结

 - 1.有了`ARC`后，程序员无需担心内存问题了。使用`ARC`来编程, 少写了很多样板代码。
 - 2.`ARC`管理对象生命周期的办法基本上是：在合适的地方插入`retain`和`release`操作。在`ARC`环境下，变量的内存管理语义可以通过修饰符指明，而原来则需要手动执行 `retain`和 `release`。
 - 3.由方法返回对象，其内存管理语义通过方法名来体现。`ARC`将此确定为开发者必须遵守的规则。
 - 4.`ARC`只负责管理OC的对象内存。尤其要注意：`CoreFoundation`对象不归`ARC`管理，开发者必须适时调用`CFRetain`和 `CFRelease`。


## ㉛ 在 dealloc方法中只释放引用并解除监听

`dealloc`方法在每个对象的生命期内仅执行一次, 也就是当保留计数降为0的时候。然而具体何时执行, 则无法保证。

### dealloc 中应该做的事情

`dealloc`方法主要用来释放对象所拥有的引用, 也就是把所有 Objective-C对象都释放掉, `ARC`会通过自动生成的 .cxx_destruct方法(参见第30条), 在 `dealloc`中为你自动添加这些释放代码。还有就是你注册的某些观察者, 也在这里进行注销。

至于对象所拥有的其他非 Objective-C对象也要释放。比如 `Core Foundation`对象就必须手工释放, 因为它们是由纯C的API所生成的。当然它们的释放时机由你来进行把控, 最好是像`MRC`一样, 不用的时候立即释放, 尽量不要等到 `dealloc`方法执行的时候再释放。还有就是程序中开销较大或系统内稀缺的资源, 比如文件描述符(file descriptor)、套接字(socket)、大块内存等, 也尽量在不使用时就立即释放。

如果对象管理着某些资源, 那么在 `dealloc`中也要调用`清理方法`, 以防开发者忘了清理这些资源。

``` swift
- (void)dealloc {
	if(!c1osed){
		NSLog(@"ERROR: close was not called before dealloc!");
		[self close];
	}
}
```

### dealloc 中要不应该做的事情

除了上述的非OC对象和开销大的资源尽量不在 `dealloc`中释放以外, 还有注意, 不要在里面随便调用其他方法。尤其是一些异步的任务, 极有可能因为回调时当前对象已经销毁而造成崩溃。

还有, 某些方法必须在特定的线程里(比如主线程里)调用才行。若在 `dealloc`里调用了那些方法, 则无法保证当前这个线程就是那些方法所需的线程。

在 `dealloc`里也不要调用属性的存取方法, 因为有人可能会覆写这些方法, 并于其中做一些无法在回收阶段安全执行的操作, 所以尽量直接使用成员变量。

此外, 在`dealloc`方法中, 尽量不要执行与当前对象有关的方法, 因为当前对象即将回收, 从而导致些莫名其妙的错误。

### 总结

v1.在 dealloc方法里, 应该做的事情就是释放指向其他对象的引用, 并取消原来订阅的`KVO`或 `NSNotificationCenter`等通知, 不要做其他事情。
 - 2.如果对象持有文件描述符等系统资源,那么应该专门编写一个方法来释放此种资源。这样的类要和其使用者约定:用完资源后必须调用 close方法。
 - 3.执行异步任务的方法不应在 `dealloc`里调用; 只能在正常状态下执行的那些方法也不应在 `dealloc`里调用, 因为此时对象已处于正在回收的状态了。


## ㉜ 编写“异常安全代码”时留意内存管理问题

Objective-C的错误模型表明, 异常只应在发生严重错误后抛出(参见第21条), 虽说如此, 不过有时仍然需要编写代码来捕获并处理异常。

在`MRC`中异常后的内存处理应该是这样的:

``` swift
    NSString *ob = [NSString new];
    @try {
        [ob initWithString:nil];
    } @catch (NSException *ex) {
        NSLog(@"%@", ex);
    } @finally {
       [ob release];
    }
```

但在`ARC`下, 默认情况下并不会帮我们释放异常后的对象内存。因为这样做需要加入大量样板代码, 以便跟踪待清理的对象, 从而在抛出异常时将其释放。可是, 这段代码会严重影响运行期的性能, 即便在不抛异常时也如此。

`-fobjc-arc-exceptions`默认是不开启的, 但通过这个编译器标志用来开启异常安全处理的功能。但有种情况编译器会自动把 `-fobjc-arc-exceptions`标志打开, 就是处于 Objective-C++模式时。因为C++处理异常所用的代码与`ARC`实现的附加代码类似, 所以令`ARC`加入自己的代码以安全处理异常, 其性能损失并不太大。

### 总结

 - 1.捕获异常时, 一定要注意将`try块`内所创立的对象清理干净。
 - 2.在默认情况下, `ARC`不生成安全处理异常所需的清理代码。开启编译器标志后, 可生成这种代码, 不过会导致应用程序变大, 而且会降低运行效率。


## ㉝ 以弱引用避免保留环

呈环状相互引用的多个对象, 将导致内存泄漏, 避免循环引用最佳的方式就是弱引用。`assign`, `unsafe_unretained`和 `weak` 都起到了弱化引用的效果。但一般前者用于基本数据类型, 后两者用于对象类型。

当属性对象回收后, `unsafe_unretained`属性仍然指向那个已经回收的实例, 而`weak`属性则会置为nil。所以使用`weak`不会令程序崩溃, 更加安全。

### 总结

 - 1.将某些引用设为`weak`,可避免出现`保留环`。
 - 2.`weak`引用可以自动清空, 也可以不自动清空。自动清空(autonilling)是随着`ARC`而引入的新特性, 由运行期系统来实现。在具备自动清空功能的弱引用上,可以随意读取其数据, 因为这种引用不会指向已经回收过的对象。


## ㉞ 以“自动释放池块”降低内存峰值

在`ARC`下不能直接使用`NSAutoreleasePool`, 而是使用`@autoreleasepool{}`, 且后者比前者效率高。而在 `MRC`下两者都是适用的。在工程创建时, 系统会自动在`main函数`中为我们创建一个自动释放池。若在没有创建自动释放池的情况下给对象发送`autorelease`消息, 则控制台会提示警告。

自动释放池可以嵌套。系统在自动释放对象时, 会把它放到最内层的池里。自动释放池机制就像`栈`(stack)一样。系统创建好自动释放池之后, 就将其推入栈中, 而清空自动释放池, 则相当于将其从栈中弹出。在对象上执行自动释放操作, 就等于将其放栈顶的那个池里。关于自动释放池原理可以参考 [Autorelease机制及释放时机](http://blog.csdn.net/wangyanchang21/article/details/51037831)。

将自动释放池嵌套用的好处是, 可以借此控制应用程序的内存峰值, 使其不致过高。比如, 在执行`for循环`时, 应用程序所占内存量就会持续上涨, 而等到所有临时对象都释放后, 内存用量又会突然下降。加上一层自动释放池之后, 应用程序在执行循环时的内存峰值就会降低, 不再像原来那么高了。

``` swift
for (int i = 0; i < 99999999; ++i) {
        @autoreleasepool {
            NSString *str = [NSString stringWithFormat:@"hi + %d", i];
            // some operation
        }
    }
```

上面代码, 手动加入自动释放池前后的内存管理对比:

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-5/20180226225027724.png" width="50%" img/>
</center>


<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-5/20180226225036788.png" width="50%" img/>
</center>


### 总结

 - 1.自动释放池排布在栈中, 对象收到 `autorelease`消息后,系统将其放入最顶端的池里。合理运用自动释放池, 可降低应用程序的内存峰值。
 - 2.`@autoreleasepool`这种新式写法能创建出更为轻便的自动释放池。


## ㉟ 用“僵尸对象”调试内存管理问题

`Cocoa`提供了`僵尸对象`(Zombie Object)这个非常方便的功能。此功能在运行期系统会把所有已经回收的实例转化成特殊的`僵尸对象`, 而不会真正回收它们。僵尸对象收到消息后, 会抛出异常, 其中准确说明了发送过来的消息, 并描述了回收之前的那个对象。

在 Xcode中开启僵尸对象调试功能的方法:
> Edit Scheme->Run->Diagnotics->Zombie objects

为了便于演示普通对象转化为僵尸对象的过程, 这段代码采用了手动引用计数。因为使用`ARC`的话,  不容易掌握对象具体的释放时机。比如下面的代码在`ARC`中并不能转化为僵尸对象, 需要通过稍微复杂些的代码才能表现出来。

``` swift
EOCClass *obj = [[EOCClass alloc]init];
NSLog(@"Bafore release:");
PrintClassInfo(obj);
[obj release];

NSLog(@"After release:");
PrintClassInfo(obj);
NSString *desc = [obj description];
```

上面的代码是一定会崩溃的, 因为向已经释放的对象发送消息了。若是开启了僵尸对象功能，那么控制台会输出下列消息：

```
Before release:   
=== EOCClass: NSObject ===   
After release:  
=== _NSZombie_EOCClass:nil===   
*** -[EOCClass description:]:message sent to
deallocated instance 0x7ff9e9c080e0   
```

这段消息明确指出了僵尸对象所收到的选择子及其原来所属的类，其中还包含接收消息的僵尸对象所对应的`指针值`。


### 僵尸对象的工作原理

`_NSZombie_EOCClass` 实际上是在运行期生成的，当首次碰到`EOCClass`类的对象要变成僵尸对象时，就会创建这么一个类。下面这段伪代码, 就演示了如何创建僵尸对象的。

``` swift
// Obtain the class of the object being deallocated
Class cls = object_getClass(self);

// Get the class's name
const char *clsName = class_getName(cls);

// Prepend _NSZombie_ to the class name
const char *zombieClsName = @"_NSZombie_" + clsName;

// See if the specific zombie class exists
Class zombieCls = objc_lookUpClass(zombieClsName);

// If the specific zombie class doesn't exists,
// then it needs to be created
if(!zombieCls){
// Obtain the template  zombie class, where the new class's 
// name is the prepended string from above
   zombieCls = objc_duplicateClass(baseZombieCls,   
   zombieClsName,0);
}

// Perform normal destruction of the object being deallocated
objc_destructInstance(self);

// Set the class of the object being deallocated
// to the zombie class
objc_setClass(self, zombieCls) 

// The class of "self" is now _NSZombie_OriginalClass
```

由于代码中的`objc_destructInstance()`函数, 对象所占内存没有释放，因此，这块内存不可复用。虽说内存泄漏了，但这只是个调试手段，所以正式发行的应用程序时一定要把这项功能关闭。

创建新类的工作由运行期函数`objc_duplicateClass()`来完成，它会把整个`NSZombie`类结构拷贝一份，并赋予其新的名字。拷贝的原因是其效率高于直接创建。

`NSZombie`类并未实现任何方法。此类为父类，因此和`NSObject`一样，也是个`根类`，该类只有一个实例变量，叫做isa。由于这个轻量级的类没有实现任何方法，所以发给它的全部消息都要经过`完整的消息转发机制`。 

### 总结

 - 1.系统在回收对象时，可以不将其真的回收，而是把它转化为僵尸对象。通过环境变量`NSZombieEnabled`可开启此功能。 
 - 2.系统会修改对象的isa 指针，令其指向特殊的僵尸类，从而使该对象变为僵尸对象。僵尸类能够响应所有的选择子，响应方式为： 打印一条包含消息内容及其接收者的消息，然后终止应用程序。


## ㊱ 不要使用 retain Count

``` swift
- (NSUInteger)retainCount;
```

它所返回的保留计数只是某个给定时间点上的值。该方法并未考虑到系统会稍后把自动释放池清空(参见第34条), 因而不会将后续的释放操作从返回值里减去, 这样的话, 此值就未必能真实反映实际的保留计数了。

而且还有一些特殊的情况:

``` swift
// 字面量语法
    NSString *string = @"abc";
    NSNumber *numberInt = @1;
    NSNumber *numberFloat = @3.141f;
    NSArray *array = @[];
    NSDictionary *dic = @{};
    NSLog(@"string:%lu", string.retainCount);
    NSLog(@"numberInt:%lu", numberInt.retainCount);
    NSLog(@"numberFloat:%lu", numberFloat.retainCount);
    NSLog(@"array:%lu", array.retainCount);
    NSLog(@"dic:%lu", dic.retainCount);
    
    [string release];
    NSLog(@"after release: %lu", string.retainCount);
```

打印结果:
```
string:18446744073709551615
numberInt:9223372036854775807
numberFloat:1
array:18446744073709551615
dic:18446744073709551615
after release: 18446744073709551615
```

上面这些都是字面量语法的使用。18446744073709551615 = 2^64 - 1，9223372036854775807 = 2^63 - 1。上面这些对象皆为`单例对象`(singleton object)，所以其保留计数都很大。

上面的字符串是个编译常量(compile-time constant), 系统会尽可能把NSStirng实现成单例对象。其它几个也类似, 以NSNumber为例，它使用了一种叫做`标签指针`(tagged pointer)的概念来标注特定类型的数值, 会把与数值有关的全部消息都放在指针值里面。运行期系统会在消息派发(参见第11条)期间检测到这种标签指针，并对它执行相应操作，使其行为看上去和真正的NSNumber对象一样。这种优化只在某些场合使用，比如范例中的浮点数对象就没有优化，所以其保留计数就是1。

另外，像刚才所说的那种单例对象，其保留计数绝对不会变。这种对象的保留及释放操作都是`空操作`(no-op)。


### 总结

 - 1.对象的保留计数看似有用，实则不然，因为任何给定时间点上的`绝对保留计数`(absolute retain count)都无法反映对象生命期的全貌。
 - 2.我们不应该总是依赖保留计数的具体值来编码。
 - 3.引入`ARC`之后，retainCount方法就正式废止了，在`ARC`下调用该方法会导致编译器报错。



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


