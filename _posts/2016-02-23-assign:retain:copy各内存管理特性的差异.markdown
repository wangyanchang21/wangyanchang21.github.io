---
title:  "assign/retain/copy各内存管理特性的差异"
date:   2016-02-23 15:47:40
categories: [iOS,内存管理]
tags: [iOS,内存管理]
---

对内存管理语句的assign、retain、copy、multableCopy等特性，做了一个差异对比和分析。

[![Contact](https://img.shields.io/badge/contact-wangyanchang21-green.svg)](https://github.com/wangyanchang21)

------

- [MRC下各内存管理特性](#mrc下各内存管理特性)
	- [assign特性](#assign特性)
	- [retain特性](#retain特性)
	- [copy特性](#copy特性)
	- [mutableCopy特性](#mutablecopy特性)
	- [关于 copy](#关于-copy)
- [各特性差异](#各特性差异)
	- [assign/retain/copy区别](#assignretaincopy区别)
	- [字符串的copy和strong](#字符串的copy和strong)
	- [容器类型的copy和strong](#容器类型的copy和strong)

------


## MRC下各内存管理特性

### assign特性

当数据类型为`int`、`float`等原生类型时，可以使用`assign`，否则可能导致内存泄露。例如当使用`malloc`分配了一块内存，并把它的地址赋值给了指针a，后来如果希望指针b也共享这块内存，于是将a赋值给(assgin)b。这时就用到了`assgin`，此时a和b指向同一块内存。

但是现在问题出现了，当a不再需要这块内存时，能都直接释放呢？肯定是不能的，因为a并不知道b是否还在使用这块内存，如果a释放了，那么b在使用这块内存的时候引起程序崩溃。

### retain特性

`retain`特性就是为了解决上述问题而提出的，使用了引用计数（reference counting），还是上面那个例子，我们给那块内存设一个引用计数，当内存被分配并且赋值给a时，引用计数是1。当把a赋值给b时引用计数增加到2。这时如果a不再使用这块内存，它只需要把引用计数减1，表明自己不再拥有这块内存。b不再使用这块内存时也把引用计数减1。当引用计数变为0的时候，代表该内存不再被任何指针所引用，系统可以直接释放掉。此时系统自动调用dealloc函数，内存被回收。

所以 retain始终是浅拷贝。引用计数每次加一。返回对象是否可变与被拷贝的对象保持一致。

### copy特性

1.对于不可变对象是浅拷贝，引用计数每次加一。始终返回一个不可变对象。  
2.对于可变对象为深拷贝，引用计数不改变。始终返回一个不可变对象。

### mutableCopy特性

始终是深拷贝，引用计数不改变。始终返回一个可变对象。

### 关于 copy

如果对一个对象使用 `copy` 必须服从 `NSCoping` 协议并且重写`copyWithZone: `方法。 字符串是系统规定的服从这个协议之后的，所以我们可以直接使用。 自定义的类则需要手动来操作了。

自定的类需要服从`<NSCopying>`协议， 举个简单的例子来说明。 自定义`Person类`的. h和. m 文件中:

``` swift
@interface Person : NSObject <NSCopying>
```

``` swift
-(id)copyWithZone:(NSZone *)zone {
    Person *per = [[Person alloc] init];
    //实例变量的拷贝
    per.name = self.name;
    per.sex = self.sex;
    per.age = self.age;
    return per;
}
```

关于深浅拷贝和完全拷贝的概念如果不太清楚，请参考[关于浅拷贝、深拷贝的探究](https://dcsnail.blog.csdn.net/article/details/51145690)。     
如果对深拷贝和完全拷贝需要透彻理解，请参考 [深拷贝和完全拷贝对比的探究](https://wangyanchang21.github.io/2016/深拷贝和完全拷贝对比的探究/)。     
这里就不再赘述了。

## 各特性差异

### assign/retain/copy区别

举个例子， 现在有两个自定义的类， 一个是 OurClass 一个是 Person， 而且 OurClass有一个类型为Person 的属性。

``` swift
//创建 OurClass 和 Person 对象
OurClass *myClass = [[OurClass alloc] init];
Person   *per = [[Person alloc] initWithName:@"蒙多" sex:@"男" age:20];

myClass.person = per;
NSLog(@"%lu",[per retainCount]);

[per release]; 
```

1.在myClass.person = per 这行:    

- 如果 Person 在属性中声明的关键字为 copy， 相当于myClass.person = [per copy];   
- 若为 retain 时，相当于myClass.person = [per retain];  
- 若为assign时，相当于直接赋值。

2.在[per release]这行:   

- 如果 Person 在属性中声明的关键字为 assign 时: 当 per释放后，myClass.person是直接赋值的，所以此时它会变成野指针。这就是为何当实例变量是对象的时候为什么不用 assign 的原因；  
- 若为 copy 时: 当 per释放后，myClass.person是新的对象，所以当销毁之前对象时，此时不会影响；  
- 若为 retain 时: 当 per释放后，myClass.person是持有那个对象的一个所有权，所以当销毁之前对象时，此时不会影响。   

### 字符串的copy和strong

我们在声明一个NSString属性时，对于其内存相关特性，在ARC下通常有两种选择：strong与copy。那这两者有什么区别呢？

``` swift
@property (nonatomic, strong) NSString *strongString;
@property (nonatomic, copy) NSString *copyedString;
```

上面的代码声明了两个字符串属性，其中一个内存特性是strong，一个是copy。

``` swift
 NSString *string = [NSString stringWithFormat:@"不可变字符串"];
    self.strongString = string;
    self.copyedString = string;
    NSLog(@"origin string: %p, %p", string, &string);
    NSLog(@"strong string: %p, %p", _strongString, &_strongString);
    NSLog(@"copy string: %p, %p", _copyedString, &_copyedString);
```

我们用一个不可变字符串来为这两个属性赋值， 输出结果为:

``` swift
origin string: 0x170446960, 0x16fd48cb8
strong string: 0x170446960, 0x10c612f78
copy string: 0x170446960, 0x10c612f80
```

这种情况下，不管是strong还是copy特性的对象，其指向的地址都是同一个，即为string指向的地址。若在MRC下，就会看到其引用计数值是3，即strong操作和copy操作都使原字符串对象的引用计数值加了1。所以， 在此情况下修饰符strong和copy是一样的效果。


如果我们把string由不可变改为可变字符串:

``` swift
NSMutableString *string = [NSMutableString stringWithFormat:@"此行改为了可变字符串"];
```

再次输出结果:

``` swift
 origin string: 0x174078080, 0x16fd61148
 strong string: 0x174078080, 0x141924a08
 copy string: 0x17424b040, 0x141924a10
```

此时copy特性字符串已不再指向string字符串对象，而是深拷贝了string字符串，并让`_copyedString`对象指向这个字符串。在MRC下，就可以看到`string`对象的引用计数是2，而`_copyedString`对象的引用计数是1。

所以，在声明NSString属性时，到底是选择strong还是copy，可以根据实际情况来定。不过，一般我们将对象声明为NSString时，都不希望它改变，所以大多数情况下，我们建议用copy，以免因可变字符串的修改导致的一些非预期问题。

但是有个问题， 就是当你避开属性的修饰符特性进行赋值时， 这个是不成立的， 因为他仅仅是赋值操作， 修饰符失效。也就是说将代码中的`self.strongString = string;`和`self.copyedString = string;` 换成 `_strongString = string;`和`_copyedString = string`。具体原因请看 [property之 self.xx与_xx的区别](http://blog.csdn.net/wangyanchang21/article/details/50607651)


### 容器类型的copy和strong

所有的容器类型，如`NSArray`、`NSDictionary`、`NSSet`以及它们的可变类`NSMutableArray`、`NSMutableDictionary`、`NSMutableSet`，在作为属性时，其修饰符应该使用copy还是strong？

与NSString类型类似，不可变类型的容器应该用`copy`，可变类型的容器应该用`strong`。

| copy  | strong  |
|:--------: | :-------------:|
| NSArray  | NSMutableArray |
| NSDictionary  | NSMutableDictionary |
| NSSet  | NSMutableSet |

与上面NSString类型的分析的原理类似。一个用`strong`来修饰不可变的容器类型的属性，为其赋值一个可变的容器类型后，该属性会`retain`这个可变容器类型，而不是`copy`。所以当修改这个可变容器你定义的不可变的容器类型的属性也会随之变化。以NSArray为例，如下面代码所示：

``` swift
@property(nonatomic, strong) NSArray *array;

NSMutableArray *mutableArray = [NSMutableArray array]；
self.array = mutableArray;
[mutableArray addObject:@"objc"];
```

那么，不可变的容器类型为什么要使用`strong`呢？

一个用`copy`来修饰不可变的容器类型的属性，为其赋值一个可变的容器类型后，该属性会`copy`这个可变容器类型。然后这时的`copy`是一个深拷贝，而且你可以使用可变类型去接收，编译器并不会报错。但返回值仍然是一个不可变容器的类型。当你一旦对该属性添加元素时，便会引起程序崩溃。

``` swift
@property (nonatomic,copy) NSMutableArray *mutableArray;

self.mutableArray = [NSMutableArray arrayWithObject:@"a"];
 [self.mutableArray addObject:@"b"];		// Crash
```


-------

欢迎指正, [wangyanchang21](https://github.com/wangyanchang21).


