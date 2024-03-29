---
title:  "+load和+initialize方法的探究"
date:   2017-08-23 16:02:49
categories: [iOS, runtime]
tags: [iOS, runtime]
---

关于+load方法和+initialize方法的联系和区别。

[![Contact](https://img.shields.io/badge/contact-wangyanchang21-green.svg)](https://github.com/wangyanchang21)

------

- [前言](#前言)
- [load的相关特点](#load的相关特点)
- [initialize的相关特性](#initialize的相关特性)
- [相关资料](#相关资料)

------


## 前言

在OC中, 根类NSObject或其子类被加载和初始化的时候，会触发一些方法，可以在适当的情况下做一些定制处理。其实, 这正是对应着`load`和`initialize`方法。

``` swift
+ (void)load;
+ (void)initialize;
```


## load的相关特点

### 运行时机
+load方法的执行时机在 App启动后, 而且是在`main函数`之前。

### load 和 runtime
当`main函数`执行前，正是从`dyld`(the dynamic link editor)到`runtime`的过程。大致过程如下：

- 系统调用当前App的进程，初始化运行环境。
- `dyld`读取并加载程序的mach-o文件和动态库。
- 开启缓存策略，加载程序中你链接的动态库，如 `framework`、`tbd`等。
- `ImageLoader`加载所有`image`（镜像）到内存。
- 当前类镜像的`map_images`函数执行。
- 当前类镜像的`load_images`函数执行，进而触发+load方法。
- 当前类镜像中所有`C++`全局静态变量和`__attribute__`修饰的构造函数的初始化。
- `main函数`执行。


### 执行顺序
一个类的+load方法不需要显式调用`[super load]`，且父类会先执行自己类中的+load方法，再执行子类的+load方法。
类的category中的+load方法也会执行，并且是先执行类后执行分类的顺序。

### 执行次数
+load方法系统只会执行一次。当子类执行自己类中的+load方法前，它的父类会先执行自己的+load方法，且仅会执行一次。+load方法不需要显式调用`[super load]`，如果在子类中手动调用`[super load]`，父类的+load才会再次执行。

### category中不会覆盖的load方法
对于某个类, 当category中重写某个方法时, 只会执行category中的此方法, 并不会执行原类中的此方法。即方法覆盖了。
+load方法在category中的使用有另外一个特性。在category中重写+load方法后并不会影响其原来类的+load方法执行, 而是如上面所说先类后分类的顺序执行。

### 继承规则特例
有个重要的事情需注意，那就是+load方法并不像普通的方法那样，它并不遵从那套继承规则。如果某个类本身没实现load方法，那么不管其各级父类是否实现此方法，系统都不会调用。所以，这就是为什么+load方法只会执行一次的原因。

### 运行环境安全
+load方法的执行时机在`main函数`之前，运行环境还是相对比较危险的。但并不能保证所有类都加载完成且可用。
但因为此时C++静态初始化程序还没有运行, 但你链接的framework在此可以确保完全加载了。

### autorelease
在+load中必要时还要自己负责做 autorelease处理。

### 影响启动时间
+load方法中的执行在App的启动过程中，任何操作都会影响App启动时长，所以尽量少的在+load方法中添加代码。

### 源码
你可以看到在运行时如何查找的+load方法作为一种特殊情况`_class_getLoadMethod`的[objc-runtime-new.mm](https://opensource.apple.com/source/objc4/objc4-532.2/runtime/objc-runtime-new.mm)，并直接调用它`call_class_loads`的[objc-loadmethod.mm](https://opensource.apple.com/source/objc4/objc4-532.2/runtime/objc-loadmethod.mm)。



## initialize的相关特性

### 运行时机
当类收到第一条消息之前初始化该类，初始化每个类只调用一次。与+load的加载时机不相同, 类似于懒加载的方式，所以也可能根本不会被调用。

### 执行顺序
一个类的+initialize方法不需要显式调用`[super initialize]`，且父类会先执行自己类或者分类中的+initialize方法，再执行子类的+initialize方法。
类的category中的+initialize方法会覆盖原类中的+initialize方法。

### 执行次数
+initialize方法系统也只会调用一次。同样不需要显式调用`[super initialize]`，如果在子类中手动调用`[super initialize]`，父类的+initialize也会再次执行。

官方文档中也明确说明，如果您想保护自己的类不被多次执行，您可以按照以下方式构建您的实现：

``` swift
+ (void)initialize {
  if (self == [ClassName self]) {
    // ... do the initialization ...
  }
}
```

### category中会覆盖的initialize方法
+initialize方法就没有+load那么特殊了，它与普通方法是一样的。当category中重写某个方法时, 只会执行category中的此方法, 并不会执行原类中的此方法了。所以就会方法覆盖了，只会执行最后一个分类中的+initialize方法的实现。

### 线程安全
运行时以线程安全的方式将初始化消息发送给类。也就是说，初始化由第一个线程运行以将消息发送到类，并且尝试向该类发送消息的任何其他线程将阻塞，直到初始化完成。所以+initialize的运行过程中是能保证线程安全的。

### 运行环境
在+initialize方法收到调用时，运行环境已经基本健全了。所以, +initialize的环境不会有较大隐患。

### 应用特点
`initialize`只应该用来设置内部数据。不应该在其中调用其他方法, 即便是本类自己的方法, 也最好别调用。若某个全局状态无法在编译期初始化, 则可以放在 `initialize`里来做。

### 源码
运行时`initialize`在`_class_initialize`函数中发送消息[objc-initialize.mm](http://opensource.apple.com/source/objc4/objc4-532.2/runtime/objc-initialize.mm)。您可以看到它用于`objc_msgSend`发送它，这是普通的消息发送功能。


最后，在`load`和`initialize`方法中尽量精简代码，在里面设置一些状态，使本类能够正常运作就可以了，不要执行那种耗时太久或需要加锁的任务。

而且在任何时候，都不要过重地依赖于这两个方法，尤其是+load方法。



## 相关资料

[Objective-C Class Loading and Initialization](https://www.mikeash.com/pyblog/friday-qa-2009-05-22-objective-c-class-loading-and-initialization.html)   
[NSObject +load and +initialize - What do they do?](https://stackoverflow.com/questions/13326435/nsobject-load-and-initialize-what-do-they-do)   

-------

欢迎指正, [wangyanchang21](https://github.com/wangyanchang21).


