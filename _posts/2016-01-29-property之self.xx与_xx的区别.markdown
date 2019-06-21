---
title:  "property之self.xx与_xx的区别"
date:   2016-01-29 15:40:47
categories: [iOS, 内存管理]
tags: [iOS, 内存管理]
---

使用属性时，关于self.xx和_xx的性能和内存优化特性对比。

[![Contact](https://img.shields.io/badge/contact-wangyanchang21-green.svg)](https://github.com/wangyanchang21)

------

- [直接区别](#直接区别)
- [深层区别](#深层区别)
- [关于->xx和_xx](#关于-xx和_xx)
- [相关资料](#相关资料)


------

关于`self.xx`和`_xx`，是同一个指针，只是前者调用该类的`setter`或`getter`方法，后者直接获取自己的实例变量。即这个问题也就演变成了属性(property)和实例变量(instance variable)的区别了。

## 直接区别

- 通过`setter`、`getter`方法和通过实例变量的区别。

`OC 2.0`之后属性一旦声明，如果没有`readonly`修饰的话，当前类自动生成了`setter`和`getter`方法的声明，并且会自动生成对应的实例变量(下划线+属性名)。而`setter`和`getter`就是访问这个实例变量的方法。

在当前类的.m文件里直接用实例变量名来访问自身的实例变量的时候，`setter`和`getter`方法是不会被调用。但是外部想用该类的实例变量就需要通过`getter`和`setter`方法了。 

## 深层区别

- 内存管理语义特性的区别

关于`setter`方法并不仅仅是将传入的参数直接赋值给实例变量，而是经过了一些简单的操作，下面是一个完整的 `setter`方法。

```swift
//属性例子
@property (nonatomic,retain)Person *person;

//setter
-(void)setPerson:(Person *)person
{  
	//判断当前传入的对象是否是已经持有的对象
    if (_person != person) {
        //释放之前的所有权
        [_person release];
        //操作 retain ,获取新的持有所有权
        _person = [person retain];
    }
}
```

很明显， 当属性的语义特性为`retain`或者`copy`的情况下，通过self.xx = **时，经过了 `setter`方法。这时实例变量会先释放，再进行 `retain` 或者 `copy` ，此时引用计数会+1，然后在赋值给实例变量。

注:用 `_xx` 访问时，在编译期程序就已经知道它的内存地址了，运行时是直接去该地址访问变量；用 `self.xx` 访问时，是在运行时通过消息机制动态的访问变量的。

`_xx` 的性能更好，但是会有一个隐患（这个隐患可能永远不会被触发），就是 OC 是非常动态的，你甚至可以在运行时添加成员变量，但是如果你添加的成员变量的内存地址在 `_xx` 的前面，那你用 `_xx` 这种硬编码的方式访问就必然会出错。

## 关于->xx和_xx

也许你也曾见过在类的外部直接取成员变量的方式，如: instance->xx。`_xx` 和 `->xx` 的方式其实都是直接取的成员变量，只是 `->` 既可以用作类的内部也可以用做类的外部。下面举个例子，有这么一个TestA类:

```swift
@interface TestA : NSObject {
@public
    NSInteger _someIvar;
    NSString *_firstName;
    NSString *_lastName;
@private
    NSInteger _age;
}

- (void)methodForTestA;
@end

@implementation TestA

- (void)methodForTestA {
    self->_someIvar = 10;
    NSInteger age = self->_age;
}

@end
```

TestA的成员变量在外部的使用:

```swift
    TestA *testA = [[TestA alloc] init];
    testA->_firstName = @"赵钱孙李";
    NSInteger someValue = testA->_someIvar;
```

还有一些情况，最好不用使用`self.xx`，具体请查看:
[在对象内部尽量直接访问实例变量](https://wangyanchang21.github.io/2018/Effective-OC%E4%B9%8B%E4%BA%94/#-%E5%9C%A8-dealloc%E6%96%B9%E6%B3%95%E4%B8%AD%E5%8F%AA%E9%87%8A%E6%94%BE%E5%BC%95%E7%94%A8%E5%B9%B6%E8%A7%A3%E9%99%A4%E7%9B%91%E5%90%AC)

## 相关资料

[属性详解(@property/@dynamic/@synthesize)](http://blog.csdn.net/wangyanchang21/article/details/50608097)  
[在对象内部尽量直接访问实例变量](https://wangyanchang21.github.io/2018/Effective-OC%E4%B9%8B%E4%BA%94/#-%E5%9C%A8-dealloc%E6%96%B9%E6%B3%95%E4%B8%AD%E5%8F%AA%E9%87%8A%E6%94%BE%E5%BC%95%E7%94%A8%E5%B9%B6%E8%A7%A3%E9%99%A4%E7%9B%91%E5%90%AC)  


-------

欢迎指正, [wangyanchang21](https://github.com/wangyanchang21).


