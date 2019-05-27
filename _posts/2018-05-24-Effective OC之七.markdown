---
title:  "高效 OC开发之系统框架"
date:   2018-05-24 16:08:16
categories: [iOS, Effective OC]
tags: [iOS, Effective OC]
---

系统框架，集合遍历，NSCache，+load和+initialize等。

[![Contact](https://img.shields.io/badge/contact-wangyanchang21-green.svg)](https://github.com/wangyanchang21)  

------

- [㊼ 熟悉系统框架](#-熟悉系统框架)
- [㊽ 多用块枚举，少用for循环](#-多用块枚举少用for循环)
- [㊾ 对自定义其内存管理语义的collection使用无缝桥接](#-对自定义其内存管理语义的collection使用无缝桥接)
- [㊿ 构建缓存时选用NSCache而非NSDictionary](#-构建缓存时选用nscache而非nsdictionary)
- [⑤① 精简initialize与load的实现代码](#-精简initialize与load的实现代码)
- [⑤② 别忘了NSTimer会持有其目标对象](#-别忘了nstimer会持有其目标对象) 

-------

## ㊼ 熟悉系统框架

将一系列代码封装为`动态库`(dynamic library)，并在其中放入描述其接口的头文件，这样做出来的东西就叫框架。有时为iOS平台构建的第三方框架所使用的是`静态库`(static library)，这是因为iOS应用程序不允许在其中包含动态库。这些东西严格来讲并不是真正的框架，然而也经常视为框架。不过，所有iOS平台的系统框架仍然使用动态库。

在为`MacOS`或`iOS`系统开发带图形界面的应用程序时，会用到名为`Cocoa框架`，在iOS上成为`Cocoa Touch`。其实`Cocoa`本身并不是框架，但是里面继承了一批创建应用程序时经常会用到的框架。

Objective-C编程时会经常需要使用底层的C语言级API。用C语言来实现API的好处是，可以绕过Objective-C的运行期系统，从而提升执行速度。

`Foundation框架`提供了基础核心功能, 除了`Foundation`与`CoreFoundation`之外，还有很多系统库:   
1.`CFNetwork`此框架提供了C语言级别的网络通信能力，它将`BSD套接字`(BSD socket)抽象成易于使用的网络接口。   
2.`CoreAudio` 该框架所提供的C语言API可用来操作设备上的音频文件。   
3.`CoreData` 此框架所提供的Objective-C接口可将对象放入数据库，便于持久保存。   
4.`CoreText` 此框架提供的C语言接口可以高效执行文字排版及渲染操作。   
5.`CoreGraphics`框架是用C语言写成的，其中提供了2D渲染所必备的数据结构与函数。   
6.`CoreAnimation`是用Objective-C语言写成的，它提供了一些工具，而UI框架则用这些工具来渲染图形并播放动画。  

还有许多其他的框架, 如`MapKit框架`，它为iOS程序提供地图功能。又比如`Social框架`，它为`MacOS`及iOS程序提供了社交网络功能。更多可以了解一下: [Cocoa Touch框架](https://blog.csdn.net/wangyanchang21/article/details/51028697#t11)。

### 总结

 - 1.许多系统框架都可以直接使用。其中最重要的是`Foundation`与`CoreFoundation`，这两个框架提供了构建应用程序所需的许多核心功能。
很多常见任务都能用框架来做，例如音频与视频处理、网络通信、数据管理等。
 - 2.请记住：用纯C写成的框架与用Objective-C写成的一样重要，若想要成为优秀的Objective-C开发者，应该掌握C语言的核心概念


## ㊽ 多用块枚举，少用for循环

在OC里面循环的语句有, `while循环`, `for循环`, `for-in快速遍历`, `NSEnumerator遍历`, `基于块的枚举遍历`。

`NSEnumerator`是个抽象基类，其中只定义了两个方法，供其具体子类来实现：

``` swift
- (NSArray*)allObjects;
- (nullable ObjectType)nextObject;
```

每次调用该方法时，其内部数据结构都会更新。等到枚举中的全部对象都已返回之后，再调用就将返回nil，这表示达到枚举末端了。

NSEnumerator遍历方式: 

``` swift
    NSArray *array = @[@1, @2, @3, @4, @5];
    NSEnumerator *enumerator = [array objectEnumerator]; // 1,2,3,4,5
    NSEnumerator *enumerator2 = [array reverseObjectEnumerator]; // 5,4,3,2,1
    
    id object;
    while ((object = [enumerator nextObject]) != nil) {
        // Do something with 'object'
    }
```

基于块的枚举遍历:

``` swift
- (void)enumerateObjectsUsingBlock:(void (NS_NOESCAPE ^)(ObjectType obj, NSUInteger idx, BOOL *stop))block;
- (void)enumerateObjectsWithOptions:(NSEnumerationOptions)options
                         usingBlock:(void(^)(id obj, NSUInteger idx, BOOL *stop))block;
```

此方式的优势：遍历时可以直接从`block`里获取更多信息。在遍历数组时，可以知道当前所针对的下标。遍历有序set(NSOrderedSet)时也一样。而在遍历字典时，无须额外编码，即可同事获取键与值。另外一个好处是，能够修改`block`的方法名，以免进行类型转换的操作，从效果上讲，相当于把本来需要执行的类型转换操作交给`block`方法签名来做。此方法还可以通过设定`stop变量`值来实现终止遍历的操作。

第一个方法是基本的遍历方法, 第二方法还可以执行反向遍历, 并发遍历。

### 总结

 - 1.遍历collection有四种方式。最基本的办法是`for循环`，其次是`NSEnumerator遍历法`及`快速遍历法`，最新、最先进的方式则是`块枚举法`。
 - 2.`块枚举法`本身就能通过`GCD`来并发执行遍历操作，无须另行编写代码。而采用其他遍历方式则无法轻易实现这一点。
 - 3.若提前知道待遍历的collection含有何种对象，则应修改块签名，指出对象的具体类型。


## ㊾ 对自定义其内存管理语义的collection使用无缝桥接

使用`无缝桥接`技术，可以在定义于`Foundation框架`中的Objective-C类和定义与`CoreFoundation框架`中的C数据结构之间互相转换。笔者将C语言级别的API称为数据结构, 而没有称其为类或对象, 这是因为它们与Objective-C中的类或对象并不相同。

转换中的`__bridge`告诉`ARC`如何处理转换所涉及的Objective-C对象。`__bridge`表示`ARC`仍然具备这个Objective-C对象的所有权。而`__bridge_retained`意味着`ARC`将交出对象的所有权。与之相似，反向转换可通过`__bridge_transfer`来实现。这三种转换方式成为`桥式转换`(bridged cast)。

`Foundation框架`中的Objective-C类所具备的某些功能，是`CoreFoundation框架`中的C语言数据结构所不具备的，反之亦然。在使用`Foundation框架`中的字典对象时会遇到一个大问题，那就是其键的内存管理语义为`拷贝`，而值的语义却是`保留`。除非使用强大的无缝桥接技术，否则无法改变其语义。

创建`CFMutableDictionary`时，可以通过下列方法来指定键和值的内存管理语义：

``` swift
CFDictionaryRef CFDictionaryCreate(
CFAllocatorRef allocator, 
const void **keys, 
const void **values, 
CFIndex numValues, 
const CFDictionaryKeyCallBacks *keyCallBacks, 
const CFDictionaryValueCallBacks *valueCallBacks);
```

下面的代码演示了这种字典的创建步骤：

``` swift
#import <Foundation/Foundation.h>
#import <CoreFoundation/CoreFoundation.h>

const void* EOCRetainCallback(CFAllocatorRef allocator,
                              const void *value)
{
    return value;
}

void EOCReleaseCallback(CFAllocatorRef allocator,
                        const void *value)
{
    CFRelease(value);
}

CFDictionaryKeyCallBacks keyCallbacks = {
    0,
    EOCRetainCallback,
    EOCReleaseCallback,
    NULL,
    CFEqual,
    CFHash
};

CFDictionaryValueCallBacks valueCallbacks = {
    0,
    EOCRetainCallback,
    EOCReleaseCallback,
    NULL,
    CFEqual
};

CFMutableDictionaryRef aCFDictionary =
        CFDictionaryCreateMutable(NULL,
                              0,
                              &keyCallbacks,
                              &valueCallbacks);
    
NSMutableDictionary *anNSDictinary =
    (__bridge_transfer NSMutableDictionary *)aCFDictionary;
```

### 总结

 - 1.通过无缝桥接技术，可以在`Foundation框架`中的Objective-C对象与`CoreFoundation框架`中的C语言数据结构之间来回转换。
 - 2.在`CoreFoundation`层面创建collection时，可以指定许多回调函数，这些函数表示此collection应如何处理器元素。然后，可运用无缝桥接技术，将其转换成具备特殊内存管理语义的Objective-C collection。


## ㊿ 构建缓存时选用NSCache而非NSDictionary

`NSCache`是`Foundation框架`专为处理缓存任务而设计的。`NSCache`胜过`NSDictionary`之处在于，当系统资源将要耗尽时，它可以自动删减缓存。此外，`NSCache`还会先行删减`最久未使用的`对象。

`NSCache`并不会`拷贝`键，而是会`保留`它。很多时候，键都是由不支持拷贝操作的对象来充当的。因此，`NSCache`不会自动拷贝键，所以说，在键不支持拷贝操作的情况下，该类用起来比字典更方便。

`NSCache`是线程安全的。而`NSDictionary`则绝对不具备此优势，在开发者自己不编写加锁代码的前提下，多个线程便可以同时访问`NSCache`。

开发者可以操控缓存删减其内容的时机。有两个与系统资源相关的尺度可供调整，其一是缓存中的对象总数，其二是所有对象的`总开销`。当对象总数或总开销超过上限时，缓存就可能会删减其中的对象了，在可用的系统资源趋于紧张时，也会这么做。

下面演示了缓存的用法：

``` swift
-(instancetype)init{
    if(self = [super init]){
        _cache = [NSCache new];
        
        //Cache a maximum of 100 URLs
        _cache.countLimit = 100;
        
        //The size in bytes of data is used as the cost
        _cache.totalCostLimit = 5*1025*1024;//5MB
    }
    return self;
}

-(void)downloadDataForURL:(NSURL *)url{
    NSData *cachedData = [_cache objectForKey:url];
    if(cachedData){
        //Cache hit
        [self useData:cachedData];
    }else{
        //Cache miss
        EOCNetworkFetcher *fetcher = [[EOCNetworkFetcher alloc]initWithURL:url];
        [fetcher startWithCompletionHandler:^(NSData *data) {
            [_cache setObject:data forKey:url cost:data.length];
            [self useData:data];
        }];
    }
}
```

还有个类叫`NSPurgeableData`和`NSCache`搭配起来用，效果很好，此类事`NSMutableData`的子类，而且实现了`NSDiscardableContent协议`。当系统资源紧张时，可以把保存`NSPurgeableData`对象的那块内存释放掉。

如果需要访问某个`NSPurgeableData`对象，可以调用其`beginContentAccess`方法，告诉它现在还不应丢弃自己所占的内存。用完之后，调用`endContentAccess`方法，告诉它在必要时可以丢弃自己所占的内存了。

刚才那个例子可以用`NSPurgeableData`改写如下：

``` swift
-(void)downloadDataForURL:(NSURL *)url{
    NSPurgeableData *cachedData = [_cache objectForKey:url];
    if(cachedData){
        //Stop the data being purged
        [cachedData beginContentAccess];
        
        //Use the cached data
        [self useData:cachedData];
        
        //Mark that the data may be purged again
        [cachedData endContentAccess];
    }else{
        //Cache miss
        EOCNetworkFetcher *fetcher = [[EOCNetworkFetcher alloc]initWithURL:url];
        [fetcher startWithCompletionHandler:^(NSData *data) {
            NSPurgeableData *purgeableData = [NSPurgeableData dataWithData:data];
            [_cache setObject:purgeableData forKey:url cost:data.length];
            
            //Don't need to beginContentAccess as it begins
            //with access already marked
            
            //Use the retrieved data
            [self useData:data];
            
            //Mark that the data may be purged now
            [purgeableData endContentAccess];
        }];
    }
}
```


### 总结

 - 1.实现缓存时应选用`NSCache`而非`NSDictionary`对象。因为`NSCache`可以提供优雅的自动删减功能，而且是`线程安全的`，此外，它与字典不同，并不会拷贝键。
 - 2.可以给`NSCache`对象设置上限，用以限制缓存中的对象总个数及`总成本`，而这些尺度则定义了缓存删减其中对象的时机。但是绝对不要把这些尺度当成可靠的`硬限制`(hard limit)，它们仅对`NSCache`起指导作用。
 - 3.将`NSPurgeableData`于`NSCache`搭配使用，可实现自动清除数据的功能，也就是说，当`NSPurgeableData`对象所占内存为系统所丢弃时，该对象自身也会从缓存中移除。
 - 4.如果缓存使用得当，那么应用程序的响应速度就能提高。只有那种`重新计算起来很费事的`数据，才值得放入缓存，比如那些需要从网络获取或从磁盘读取的数据。


## ⑤① 精简initialize与load的实现代码

### + load;

对于加入运行期系统中的每个类及分类来说，必定会调用此方法，而且仅调用一次。当包含类或分类的程序库载入系统时，就会执行此方法，而这通常就是指应用程序启动的时候，若程序是为iOS平台设计的，则肯定会在此时执行。如果类和其分类中都定义了`load方法`，则先调用类里的，再调用分类里的。

`load`的运行环境还是相对比较危险的, 因为运行期的系统可能还不完整。而且无法判断出其中各个类的载入顺序, 所以在`load方法`中使用其他类是不安全的。

在执行子类的`load方法`之前，必定会先执行所有超类的`load方法`，所以在`load方法`里不用写[super load]。而如果代码还依赖了其他程序库，那么程序库里相关类的`load方法`也必定会先执行。

有个重要的事情需注意，那就是`load方法`并不像普通的方法那样，它并不遵从那套继承规则，如果某个类本身没实现`load方法`，那么不管其各级超类是否实现此方法，系统都不会调用。

而且load方法务必实现得精简一些，也就是要尽量减少其所执行的操作，因为整个应用程序在执行`load方法`时都会阻塞。


### + initialize;

对于每个类来说，该方法会在程序首次用该类之前调用，类似于懒加载的方式，且只调用一次。它是由运行期系统来调用的，绝不应该通过代码直接调用。

从运行期系统完整度上来讲，此时可以安全使用并调用任意类中的任意方法。而且运行期系统也能保证`initialize方法`一定会在`线程安全的环境`中执行。

`initialize方法`与其他方法(load除外)一样，也遵循通常的继承规则。如果某个类未实现它，而其超类实现了，那么就会运行超类的实现代码。如果超类要保护自己不要多次运行，可以按照以下方式构建您的实现：

``` swift
+ (void)initialize {
  if (self == [ClassName class]) {
    // ... do the initialization ...
  }
}
```

`initialize方法`只应该用来设置内部数据。不应该在其中调用其他方法, 即便是本类自己的方法, 也最好别调用。若某个全局状态无法在编译期初始化, 则可以放在 `initialize`里来做。

在`load`和`initialize`方法中尽量精简代码，在里面设置一些状态，使本类能够正常运作就可以了，不要执行那种耗时太久或需要加锁的任务。

之前也曾写过一篇关于`load`和`initialize`方法比较的博客: [NSObject的load和initialize方法比较](https://blog.csdn.net/wangyanchang21/article/details/77483749)。

### 总结 

 - 1.在加载阶段，如果类实现了`load方法`，那么系统就会调用它。分类里也可以定义此方法，类的`load方法`要比分类中的先调用。与其他方法不同，`load方法`不参与覆写机制。
 - 2.首次使用某个类之前，系统会向其发送`initialize`消息。由于此方法遵从普通的覆写规则，所以通常应该在里面判断当前要初始化的是哪个类。
 - 3.`load`与`initialize`方法都应该实现得精简一些，这有助于保持应用程序的响应能力，也能减少引入`依赖环`(interdependency cycle)的几率。
 - 4.无法在编译器设定的全局变量，可以放在`initialize`方法里初始化。


## ⑤② 别忘了NSTimer会持有其目标对象

计时器要和 `运行循环`(run loop)相关联，运行循环到时候会触发任务。创建 `NSTimer` 时，可以将其`预先安排`在当前的运行循环中，也可以先创建好，然后由开发者自己来调度。无论采用哪种方式，只有把计时器放在运行循环里，它才能正常触发任务。

创建计时器的时候，由于目标对象是 self ，所以要保留此实例。然而，因为计时器是用实例变量存放的，所以实例也保留了计时器。于是，就产生了 `保留环`，如果此环能在某一时刻打破，那就不会出什么问题。然而要想打破保留环，只能将计时器置为nil或使计时器无效。

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-7/20180524154317173.png" width="70%" img/>
</center>

但还有种比较巧妙的化解方案:

``` swift
@implementation NSTimer (EOCBlocksSupport)

+ (NSTimer *)eoc_scheduledTimerWithTimeInterval:(NSTimeInterval)interval block:(void(^)())block repeats:(BOOL)repeats {
　　return [self scheduledTimerWithTimeInterval:interval target:self selector:@selector(eoc_blockInvoke:) userInfo: [block copy] repeats:repeats];
}

+ (void)eoc_blockInvoke:(NSTimer *)timer {
　　void (^block)() = timer.userInfo;
　　if (block) {
　　　　block();
　　}
}
@end
```

``` swift
- (void)fireCounting {
	__weak EOCClass *weakSelf = self;
	_pollTimer = [NSTimer eoc_scheduledTimerWithTimeInterval: 5.0 block:^{
		EOCClass *strongSelf = weakSelf;
	    [strongSelf counting];
    }
     repeats:YES];
 }
```

上面这个方法不用手动进行对timer的干预从而做到避开循环引用的问题, 原理就是`weak化`。

<center>
<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/effectiveoc-7/20180524160353342.png" width="70%" img/>
</center>



### 总结

 - 1.`NSTimer`对象会保留其目标，直到计时器本身失效为止，调用`invalidate方法`可令计时器失效，另外，一次性的计时器在触发完任务之后也会失效。
 - 2.反复执行任务的计时器，很容易引入保留环，如果这种计时器的目标对象又保留了计时器本身，那肯定会导致保留环。这种环状保留关系，可能是直接发生的，也可能是通过对象图里的其他对象间接发生的。
 - 3.可以扩充`NSTimer`的功能，用`块`来打破保留环。不过，除非`NSTimer`将来在公共接口里提供此功能，否则必须创建分类，将相关实现代码加入其中。




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


