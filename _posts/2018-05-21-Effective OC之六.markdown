---
title:  "高效 OC开发之Block和GCD"
date:   2018-05-21 12:53:04
categories: [iOS, Effective OC]
tags: [iOS, Effective OC]
---

block, GCD，NSOperationQueue等实现多线程。

[![Contact](https://img.shields.io/badge/contact-wangyanchang21-green.svg)](https://github.com/wangyanchang21)  

------

**高效 OC开发系列文章：**   
[高效 OC开发之熟悉Objective-C](https://wangyanchang21.github.io/2017/Effective-OC%E4%B9%8B%E4%B8%80)  
[高效 OC开发之对象、消息、运行时](https://wangyanchang21.github.io/2017/Effective-OC%E4%B9%8B%E4%BA%8C)  
[高效 OC开发之接口与API设计](https://wangyanchang21.github.io/2017/Effective-OC%E4%B9%8B%E4%B8%89)  
[高效 OC开发之协议与分类](https://wangyanchang21.github.io/2018/Effective-OC%E4%B9%8B%E5%9B%9B)  
[高效 OC开发之内存管理](https://wangyanchang21.github.io/2018/Effective-OC%E4%B9%8B%E4%BA%94)  
[高效 OC开发之Block和GCD](https://wangyanchang21.github.io/2018/Effective-OC%E4%B9%8B%E5%85%AD)  
[高效 OC开发之系统框架](https://wangyanchang21.github.io/2018/Effective-OC%E4%B9%8B%E4%B8%83)  

-------

## ㊲ 理解"块"的概念

`block`和函数类似, 只不过是直接定义在另一个函数里的, 和定义它的那个函数共享同一个范围内的东西。`block`可以实现闭包, 有些人也称它作块。而且, iOS多线程的核心就是`block`和`GCD`(Grand Central Dispatch)。

### __block

在默认情况下, `block`捕获的变量是不可以在`block`内部进行修改的。若想修改捕获的变量需要加`__block`进行修饰。 

### block类型

`block`其实会按照存储位置进行分类, 在`MRC`中, 可能有三种`block`, 就是全局块, 栈块和堆块。 但是在`ARC`中, 只有两种`block`了, 就是全局块和堆块了。由于`ARC`已经能很好地处理对象的生命周期的管理, 所以都放到堆上管理, 不在使用栈块管理了, 所以就没有栈块的。 

但是有一种例外情况，就是所创建的`block`没有被外部指针所持有的时候，编译器就不会做出将其拷贝的堆区的操作。所以这种情况下，是存在`栈块`的。


### block 的内部结构和作用

`block`是个什么东西呢, 对象? 结构体? 让我们来看一下`block`的内部结构:

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-6/20180312115212317.png" width="70%" img/>
</center>


``` swift
struct Block_layout {
    void *isa;
    int flags;
    int reserved; 
    void (*invoke)(void *, ...);
    struct Block_descriptor *descriptor;
    // imported variables
};

struct Block_descriptor {
    unsigned long int reserved;
    unsigned long int size;
    void (*copy)(void *dst, void *src);
    void (*dispose)(void *);
};
```

通过上面的结构, 可以看出一个 `block` 实例的构成实际上有6个部分：  
1.`isa指针`: 所有对象都有该指针，用于实现对象相关的功能。   
2.`flags`: 附加标识位, 在`copy`和`dispose`等情况下可以用到。   
3.`reserved`:保留变量。   
4.`invoke`: 函数指针，指向 `block`的实现代码, 也可以说是函数调用地址。   
5.`descriptor`:  表示该 `block`的附加描述信息，主要是 `size`，以及 `copy`和 `dispose`函数的指针。这两个辅助函数在拷贝及丢弃块对象时运行, 其中会执行一些操作, 比方说, 前者要保留捕获的对象,而后者则将之释放。   
6.`variables`: 捕获的变量，`block`能够访问它外部的局部变量，就是因为将这些变量复制到了结构体中。  

`block`的结构体中是有isa指针的, 它还有引用计数, 而且还能响应选择子, 所以可视为对象。这里就不详述了, 因为之前也写过了关于`block`的博客: [浅谈block实现原理及内存特性](https://blog.csdn.net/wangyanchang21/article/details/79525394)。


### 总结

 - 1.块是C、C++、 Objective-C中的词法闭包。
 - 2.块可接受参数, 也可返回值。
 - 3.块可以分配在栈或堆上,也可以是全局的。分配在栈上的块可拷贝到堆里, 这样的话, 就和标准的 Objective-C对象一样, 具备引用计数了。


## ㊳ 为常用的块类型创建 typedef

每个块都具备其`固有类型`(inherent type), 因而可将其赋给适当类型的变量。
为了隐藏复杂的块类型,需要用到C语言中名为`类型定义`(type definition)的特性。typedef关键字用于给类型起个易读的别名。使用类型定义还有个好处,就是当你打算重构块的类型签名时会很方便。

``` swift
typedef void (^actionBlock)(int cardId);
```

最好在使用块类型的类中定义这些 `typedef`,而且还应该把这个类的名字加在由 `typedef`所定义的新类型名前面,这样可以阐明块的用途。还可以用 `typedef`给同一个块签名类型创建数个别名。在`Accounts框架`中就有这样的例子:

``` swift
typedef void(^ACAccountStoreSaveCompletionHandler)(BOOL success, NSError *error);
typedef void(^ACAccountStoreRemoveCompletionHandler)(BOOL success, NSError *error);
```

### 总结

 - 1.以`typedef`重新定义块类型, 可令块变量用起来更加简单。
 - 2.定义新类型时应遵从现有的命名习惯, 勿使其名称与别的类型相冲突。
 - 3.不妨为同一个块签名定义多个类型别名。如果要重构的代码使用了块类型的某个别名, 那么只需修改相应 `typed`中的块签名即可, 无须改动其他 `typedef`。


## ㊴ 用handler块降低代码分散程度

异步方法在执行完任务之后, 需要以某种手段通知相关代码。实现此功能有很多办法。常用的技巧是委托协议(参见第23条), `block`和通知等方式。常用的代理协议代码的分散度比较高, 且若在当前类中有多个`delegate`的话, 还需要在代理回调中进行判断。

与使用委托模式的代码相比, 用块写出来的代码显然更为整洁。异步任务执行完毕后所需运行的业务逻辑, 和启动异步任务所用的代码放在了一起。而且, 由于块声明在创建获取器的范围里, 所以它可以访问此范围内的全部变量。

有时候会成功和失败的情况要分别处理, 所以调用此API的代码也就会按照逻辑, 把应又对成功和失败情况的代码分开来写, 这将令代码更易读懂。API格式如下:

``` swift
- (void)startRequestWithSuccessBlock:(void (^)(id data))success
					    failureBlock:(void (^)(id data))failure;
```

而且, 若有需要, 还可以把处理失败情况或成功情况所用的代码省略。

``` swift
- (void)startRequestWithHandelBlock:(void (^)(id data))handel;
```

把成功情况和失败情况放在同一个块中, 有些缺点, 就是由于全部逻辑都写在一起, 所以会令块变得比较长, 且比较复杂。然而只用一个块的写法也有好处, 那就是更为灵活。而且, 在调用API的代码可能会在处理成功响应的过程中发现错误。

基于 `handler`来设计API还有个原因, 就是某些代码必须运行在特定的线程上。比方说, `Cocoa`与 `Cocoa Touch`中的UI操作必须在主线程上执行。这就相当于`GCD`中的`主队列`(main queue)。因此, 最好能由调用API的人来决定 `handler`应该运行在哪个线程上。

``` swift
- (void)doSomeThingOnQueue:(NSOperationQueue *)queue
			   actionBlock:(void (^)(id data))handel;
```

### 总结

 - 1.在创建对象时,可以使用内联的 `handler块`将相关业务逻辑一并声明。
 - 2.在有多个实例需要监控时, 如果采用委托模式,那么经常需要根据传入的对象来切换, 而若改用 `handler块`来实现, 则可直接将块与相关对象放在一起。
 - 3.设计API时如果用到了 `handler块`, 那么可以增加一个参数, 使调用者可通过此参数来决定应该把块安排在哪个队列上执行。


## ㊵ 用块引用其所属对象时不要出现保留环

使用`block`时很容易已发循环引用的问题。中呈环状相互引用的多个对象, 将导致内存泄漏, 因为循环中的对象其保留计数不会降为0。所以, 通常通过`弱引用`(weak reference, 参见第33条), 或是从外界命令循环中的某个对象不再保留另外一个对象。从而打破保留环, 避免内存泄漏。

### 总结 

 - 1.如果块所捕获的对象直接或间接地保留了块本身, 那么就得当心保留环问题。
 - 2.一定要找个适当的时机解除保留环, 而不能把责任推给API的调用者。


## ㊶ 多用派发队列, 少用同步锁

在 Objective-C中, 如果有多个线程要执行同一份代码, 那么有时可能会出问题。这种情况下, 通常要使用锁来实现某种同步机制。

#### 锁

1.`同步锁`(synchronization block)

``` swift
- (void)synchronizationMethod {
    @synchronized(self) {
        // Safe Code
    }
}
```

滥用`@synchronized(self)`则会降低代码效率, 因为共用同一个锁的那些同步块,都必须按顺序执行。若是在self对象上频繁加锁, 那么程序可能要等另一段与此无关的代码执行完毕, 才能继续执行当前代码, 这样做其实并没有必要。

2.`NSLock`:

``` swift
- (void)synchronizationMethod {
    NSLock *lock = [[NSLock alloc] init];
    [lock lock];
    // Safe Code
    [lock unlock];
}
```

这两种方法都很好, 不过也有其缺陷。比方说, 在极端情况下, 同步块会导致死锁, 另外, 其效率也不见得很高, 而如果直接使用锁对象的话, 一旦遇到死锁, 就会非常麻烦。

3.`递归锁`(NSRecursiveLock)

所有还有一种锁叫递归锁, 将`NSLock`改为`NSRecursiveLock`后, 线程能够多次持有该锁, 而且不会出现死锁的现象。

#### GCD队列

有种简单而高效的办法可以代替同步块或锁对象, 那就是使用`串行同步队列`( serial synchronization queue), 它是一种轻量级的机制。将读取操作及写人操作都安排在同一个队列里, 即可保证数据同步。


| 任务派发方式 | 说明 |
| :-----: | :-----: |
| dispatch_sync() | 同步执行，完成了它预定的任务后才返回，阻塞当前线程 |
| dispatch_async() | 异步执行，会立即返回，预定的任务会完成但不会等它完成，不阻塞当前线程 |


| GCD队列种类 | 获取方法 | 队列类型 | 说明 |
| :-----: | :-----: | :-----: | :-----: |
| 主队列 | dispatch_get_main_queue | 串行队列 | 主线中执行 |
| 全局队列 | dispatch_get_global_queue | 并发队列 | 子线程中执行 |
| 用户队列 | dispatch_queue_create | 串行/并发 | 子线程中执行 |


使用`GCD`队列替换锁的方式, 把数据写入操作与数据读取操作都安排在序列化的队列里执行:

``` swift
_syncQueue = dispatch_queue_create("com.effetiveobjectivec.syncQueue", NULL);

- (NSString *)someString {
    __block NSString *localSomeString;
    dispatch_sync(_syncQueue, ^{
        localSomeString = _someString;
    });
    return _someString;
}

- (void)setSomeString:(NSString *)someString {
    dispatch_sync(_syncQueue, ^{
        _someString = someString;
    });
}
```

而且我们还可以进一步优化。数据写入不一定非得是同步的。设置实例变量所用的块，并不需要向设置方法返回什么值。那代码可以改成：

``` swift
- (void)setSomeString:(NSString *)someString {
    dispatch_async(_syncQueue, ^{
        _someString = someString;
    });
}
```

这只是把同步派发改成了异步派发, 从调用者的角度来看, 这个小改动可以提升设置方法的执行速度, 而读取操作与写入操作依然会按顺序执行。

但这么改有个问题需要注意: 因为执行异步派发时, 需要拷贝块。若拷贝块所用的时间明显超过执行块所花的时间, 则这种做法将比原来更慢。由于本书所举的这个例子很简单, 所以改完之后很可能会变慢。然而, 若是派发给队列的块要执行更为繁重的任务, 那么仍然可以考虑这种备选方案。

先引入栅栏(barrier)的概念:

``` swift
// 如果传入自己创建的并行队列时，会阻塞当前队列执行，而不阻塞当前线程。
void dispatch_barrier_async(dispatch_queue_t queue, dispatch_block_t block);
// 如果传入自己创建的并行队列时，阻塞当前队列的同时也会阻塞当前线程。
void dispatch_barrier_sync(dispatch_queue_t queue, dispatch_block_t block);
```

再次优化, 使用`GCD`并发队列和栅栏(barrier):

``` swift
_syncQueue = dispatch_queue_create("com.effetiveobjectivec.syncQueue", DISPATCH_QUEUE_CONCURRENT);

- (NSString *)someString {
    __block NSString *localSomeString;
    dispatch_sync(_syncQueue, ^{
        localSomeString = _someString;
    });
    return localSomeString;
}

- (void)setSomeString:(NSString *)someString {
    dispatch_barrier_async(_syncQueue, ^{
        _someString = someString;
    });
}
```

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-6/20180517160524947.png" width="50%" img/>
</center>

在这个并发队列中，读取操作是用普通的块来实现的，而写入操作则是用栅栏块来实现的 读取操作可以并行，但写入操作必须单独执行，因为它是栅栏块。

所以, 测试一下性能，你就会发现，这种做法肯定比使用串行队列要快。

### 总结

 - 1.派发队列可用来表述同步语义(synchronization semantic), 这种做法要比使用`@synchronized()`或`NSLock`对象更简单。
 - 2.将同步与异步派发结合起来, 可以实现与普通加锁机制一样的同步行为, 而这么做却不会阻塞执行异步派发的线程。
 - 3.使用同步队列及栅栏块, 可以令同步行为更加高效。


## ㊷ 多用GCD, 少用performSelector系列方法

`performSelector`系列方法有很多, 都是带有选择子的。这种编程方式极为灵活，经常可用来简化复杂的代码。不管哪种用法，编译器都不知道要执行的选择子是什么，这必须到了运行期才能确定。

这种方式的确定很明显。编译器并不知道将要调用的选择子是什么，因此也就不了解其方法签名及返回值，甚至连是否有返回值都不清楚。而且，由于编译器不知道方法名，所以就没办法运用`ARC`的内存管理规则来判定返回值是不是应该释放，鉴于此，`ARC`采用了比较谨慎的做法，就是不添加释放操作。然而这么做可能导致内存泄漏，因为方法在返回对象时可能已经将其保留了。具体的例子可以阅读我另一篇博客, [ARC 不会优化的情景](https://blog.csdn.net/wangyanchang21/article/details/79461511#t22)。
再有, 这些方法的返回值只能是void或者对象类型(id类型), 局限性很大。

再举个例子, `performSelector`还有如下几个版本，可以再发消息时顺便传递参数:

``` swift
- (id)performSelector:(SEL)aSelector withObject:(id)object;
- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;
```

但其实局限颇多。由于参数类型是`id`，所以传入的参数必须是对象才行。此外，选择子最多只能接受两个参数，而在参数不止两个的情况下，则没有对应的`performSelector`方法能够执行此种选择子。只能打包更多参数进入集合中再传递。

所以, 要避免使用`performSelector`系列方法所提供的线程功能，因为这些功能都可以通过在大中枢派发机制中使用块来实现。延后执行可以用`dispatch_after`来实现，在另一个线程上执行任务则可通过`dispatch_sync`及`dispatch_async`来实现。


### 总结 

 - 1.`performSelector`系列方法在内存管理方面容易有疏失。它无法确定将要执行的选择子具体是什么，因而`ARC`编译器也就无法插入适当的内存管理方法。
 - 2.`performSelector`系列方法所能处理的选择子太过局限了，选择子的返回值类型及发送给方法的参数个数都受到限制。
 - 3.如果想把任务放在另一个线程上执行，那么最好不要用`performSelector`系列方法，而是应该把任务封装到块里，然后调用大中枢派发机制的相关方法来实现。



## ㊸ 掌握GCD及操作队列的使用时机

`GCD`是纯C的API，而`操作队列`(NSOperationQueue)则是Objective-C的API, 而且操作队列在底层是用`GCD`来实现的。在`GCD`中，任务用块来表示，而块是个轻量级数据结构。与之相反，`操作`(NSOperation)则是个更为重量级的Objective-C对象。

在执行后台任务时，`GCD`并不一定是最佳方式, 操作队列有很多地方胜过派发队列。使用`NSOperation`及`NSOperationQueue`的好处如下：   
1. `NSOperation`可以取消某个操作, 而`GCD`没有取消操作。如果使用操作队列，那么想要取消操作是很容易的。    
2. `NSOperation`可以指定优先级, 而`GCD`只支持FIFO队列。操作的优先级表示此操作与队列中的其他操作之间的优先级关系。    
3. `NSOperationQueue`支持在操作之间设置依赖关系, 而`GCD`没有内建的依赖关系支持。一个操作可以依赖其他多个操作。开发者能够指定操作之间的依赖体系，使特定的操作必须在另外一个操作顺利执行完毕后方可执行。   
4. `NSOperationQueue`秉容`KVO`)。`NSOperation`对象有很多属性都适合通过KVO来进行监测, 这意味着你可以观察任务的状态。   
5. `NSOperation`可以自定义子类。除了系统内置的子类，还可以自定义`NSOperation`的子类。这些类就是普通的 Objective-C对象, 能够存放任何信息, 还可以随意调用定义在类中的方法。这就比派发队列中那些简单的块要强大许多。   

那我们应该只用`NSOperationQueue`而不用`GCD`吗? 答案是否定的。 因为`NSOperationQueue`的执行速度比`GCD`慢。`NSOperation`和 `GCD`并不相互排斥。你可以把复杂的任务交于`NSOperationQueue`去处理, 而把简单的任务交于`GCD`去处理, 能在两者之间的结合使用会使你的程序更高效, 更强大。

iOS多线程还有NSThread, 它的缺点是需要手动管理所有的线程活动, 而且执行方法都是通过performSelector来完成的。 所以需要等到运行时才能确定, 且可能导致内存泄漏, 具体原因请看本文的第㊷条。但是有一点, `GCD`和`NSOperationQueue`不需要操心任务在哪条线程上处理, 因为系统会做出最优化线程选择。然而NSThread能准确的指定线程, 在某个线程上执行任务。

### 总结

 - 1.在解决多线程与任务管理问题时，派发队列并非唯一方案。
 - 2.操作队列提供了一套高层的Objective-C API，能实现纯`GCD`所具备的绝大部分功能，而且还能完成一些更为复杂的操作，那些操作若改用`GCD`来实现，则需另外编写代码。
 - 3.根据实际情况来选择多线方式, `NSThread`、`GCD`还是`NSOperationQueue`。


## ㊹ 通过Dispatch Group机制, 根据系统资源状况来执行任务

`dispatch group`(派发分组, 调度组)是`GCD`的一项特性，能够把任务分组。调用者可以等待这组任务执行完毕，也可以在提供回调函数之后继续往下执行，这组任务完成时，调用者会得到通知。

它可以把一些任务归入一个组内来执行，并通过监听组内所有任务的总体完成情况来做下一步相应处理。一般通过`dispatch_group_async`把块内的任务添加进`group`中, 也有手动方法`dispatch_group_enter`、`dispatch_group_leave`。

任务添加后, 有两个方法可以关联执行:   
`dispatch_group_wait`: 同步等待当前任务组执行完毕, 完毕后解除线程阻塞。当前任务组执行时间超出timeout时或者任务组完成时，该函数返回。可以传入的`timeout`参数设定等待时间, 表示阻塞多久。官方还提供`DISPATCH_TIME_NOW`和`DISPATCH_TIME_FOREVER`常数方便使用。   
`dispatch_group_notify`: 待任务组执行完毕时调用，不会阻塞当前线程。等待任务组执行完毕之后，块会在特定的线程上执行。   

从`Dispatch Group`机制, 我们也可以看出资源配置的问题。为了执行队列中的块，`GCD`会在适当的时机自动创建新线程或复用旧线程。如果使用并发队列，那么其中有可能会有多个线程，这也就意味着多个块可以并发执行。在并发队列中，执行任务所用的并发线程数量，取决于各种因素，而`GCD`主要是根据系统资源状况来判断这些因素的。由于`GCD`有并发队列机制，所以能够根据可用的系统资源状况来并发执行任务。

一个关于循环的函数`dispatch_apply`: 此函数会将块反复执行一定的次数，每次传给块的参数值都会递增。`dispatch_apply`如果使用串行队列就类似我们平时缩写的`for循环`, 所以意义不大。如果采用并发队列，那么系统就可以根据资源状况来并行执行这些块了

### 总结

 - 1.一系列任务可归入一个`dispatch group`之中。开发者可以在这组任务执行完毕时获得通知。
 - 2.通过`dispatch group`，可以在并发式派发队列里同时执行多项任务。此时`GCD`会根据系统资源状况来调度这些并发执行的任务。


## ㊺ 使用dispatch_once来执行只需执行一次的线程安全代码

使用`dispatch_once`可以简化代码并且彻底保证线程安全，开发者根本无须担心加锁或同步。所有问题都由`GCD`在底层处理。

`dispatch_once`更高效，它没有使用重量级的同步机制。此函数采用`原子访问`(atomic access)来查询标记，以判断其所对应的代码原来是否已经执行过。所以使用它来替代同步锁的话, 速度可以提前一倍。


### 总结

 - 1.经常需要编写`只需执行一次的线程安全代码`(thread-safe single-code execution)。通过`GCD`所提供的`dispatch_once`函数，很容易就能实现此功能。
 - 2.标记应该声明在`static`或`global`作用域中，这样的话，在把只需执行一次的块传给`dispatch_once`函数时，传进去的标记也是相同的。


## ㊻ 不要使用dispatch_get_current_queue

使用`GCD`时，经常需要判断当前代码正在哪个队列上执行，向多个队列派发任务时，更是如此。`dispatch_get_current_queue`函数返回当前正在执行代码的队列，不过用的时候要小心。从iOS系统6.0版本起，已经将其废弃了。

该函数有种典型的错误用法(antipattern，`反模式`)，就是用它检测当前队列是不是某个特定的队列，试图以此来避免执行同步派发时可能遭遇的死锁问题。

``` swift
if(dispatch_get_current_queue()==queueA){
		// Code1
 }else{
		// Code2
}
```

使用队列时还要注意另外一个问题，而那个问题会在你意想不到的地方导致死锁。队列之间会形成一套层级体系，这意味着排在某条队列中的块，会在其上级队列(parent queue，也叫`父队列`)里执行。层级里地位较高的那个队列总是`全局并发队列`。由于队列间有层级关系，所以`检查当前队列`是否为执行同步派发所用的队列这种办法，并不总是奏效。

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-6/20180521114854523.png" width="50%" img/>
</center>

使用这种API的开发者可能误以为：在回调块里调用`dispatch_get_current_queue`所返回的`当前队列`，总是其调用API时指定的那个。但实际上返回的却是API内部的那个同步队列。

要解决这个问题，最好的办法就是通过`GCD`所提供的功能来设定`队列特有数据`(queue-specific data)，此功能可以把任意数据以键值对的形式关联到队列里。最重要之处在于，假如根据指定的键获取不到关联数据，那么系统就会沿着层级体系向上查找，直至找到数据或到达根队列为止。

``` swift
static int kQueueSpecific;
CFStringRef queueSpecificValue = CFSTR("queueA");
dispatch_queue_set_specific(queueA,
                            &kQueueSpecific,
                            (void*)queueSpecificValue,
                            (dispatch_function_t)CFRelease);


CFStringRef retrievedValue =
	dispatch_get_specific(&kQueueSpecific);
if(retrievedValue){
	// Code1
 }else{
	// Code2
}
```

最后要说明的是, 并不是说`dispatch_get_current_queue`就完全没有可用之地。其官方文档中写道, 它建议使用于仅限于调试的环境下。在此情况下，可以放心使用这个已经废弃的方法，只是别把它编译到发行版的程序里就行。


### 总结

 - 1.`dispatch_get_current_queue`函数的行为常常与开发者所预期的不同。此函数已经废弃，只应做调试之用。
 - 2.由于派发队列是按层级来组织的，所以无法单用某个队列对象来描述`当前队列`这一概念。
 - 3.`dispatch_get_current_queue`函数用于解决由不可重入的代码所引发的死锁，然而能用此函数解决的问题，通常也能改用`队列特定数据`来解决。


-------

欢迎指正, [wangyanchang21](https://github.com/wangyanchang21).


