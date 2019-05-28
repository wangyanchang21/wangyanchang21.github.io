---
title:  "Architectures与指令集架构"
date:   2016-10-26 16:29:59
categories: [iOS, 指令集架构]
tags: [iOS，指令集架构]
---

关于iOS设备中CPU架构或指令集架构的介绍，如armv6, armv7, armv7s, arm64, i386, x86_64。

[![Contact](https://img.shields.io/badge/contact-wangyanchang21-green.svg)](https://github.com/wangyanchang21)

------

- [简介](#简介)
- [iPhone的CPU](#iphone的cpu)
	- [iPhone真机的CPU架构](#iphone真机的cpu架构)
	- [iPhone模拟器的CPU架构](#iphone模拟器的cpu架构)
- [Xcode中配置 Architectures](#xcode中配置-architectures)
	- [Architectures](#architectures)
	- [Valid architectures](#valid-architectures)
	- [Build Active Architecture Only](#build-active-architecture-only)
	- [注意事项](#注意事项)

------

## 简介

前些天遇到的关于iPhone模拟器CPU架构不支持的问题, 今天来彻底的说一说关于iPhone真机和模拟器的CPU架构或指令集架构。还要说一说`Xcode`中`BuildSetting`中的`Architectures`, `Valid architectures`, `Build Active Architecture Only`的配置项。

## iPhone的CPU

`Arm处理器`，因为其低功耗和小尺寸而闻名，几乎所有的手机处理器都基于`arm`，其在嵌入式系统中的应用非常广泛，它的性能在同等功耗产品中也很出色。苹果手机使用的CPU是自主研发的,所用指令集架构`arm`公司的。
iPhone7搭载64位第三代的A10处理器,iPhone6s, iPhone SE搭载64位第二代的A9处理器,iPhone6搭载64位第一代的A8处理器, iPhone5s处理器架构是64位`armv8`。 其实iPhone 5s用的指令集架构其实是64位的`armv8`, 从iPhone 6开始(即从苹果自主研发第一代的A8开始) iphone手机的架构都是`arm64`了, 具体的是什么指令集就不得而知了。 

### iPhone真机的CPU架构

iPhone的CPU架构一般是指iPhone手机CPU的指令集架构。 `armv6`、`armv7`、`armv7s`、`arm64`都是arm处理器的指令集，所有指令集原则上都是向下兼容的，如iPhone4S的CPU默认指令集为`armv7`指令集，但它同时也兼容`armv6`指令集，只是使用`armv6`指令集时无法充分发挥其性能，即无法使用`armv7`指令集中的新特性，同理，iPhone5的处理器标配`armv7s`指令集，同时也支持`armv7`指令集，只是无法进行相关的性能优化，从而导致程序的执行效率没那么高。
从iPhone 5s开始就摒弃了32位CPU架构了, 而是采用了64位CPU架构。 

- `arm64`: iPhone 5s, iPhone 6(Plus), iPhone 6s(Plus), iPhone SE, iPhone 7(Plus), iPad Air(2), Retina iPad Mini(2,3)……   
- `armv7s`: iPhone 5, iPhone 5c, iPad 4    
- `armv7`: iPhone 3GS, iPhone 4, iPhone 4S, iPod 3G/4G/5G, iPad, iPad 2, iPad 3, iPad Mini     
- `armv6`: iPhone, iPhone 3G, iPod 1G/2G   

### iPhone模拟器的CPU架构

模拟器32位处理器测试需要`i386`架构，iPhone5及之前设置:`i386`。

模拟器64位处理器测试需要`x86_64`架构，iPhone5s及之后设备。

前阵子QQ分享不在支持`i386`架构了, 所以造成运行iphone5以下的模拟器时, 会报 “Undefined symbols for architecture”的错误, 原因就如此。 具体的看已查看我的另一篇博客: [错误总结:Undefined symbols for architecture](http://blog.csdn.net/wangyanchang21/article/details/51425309)


## Xcode中配置 Architectures

在Xcode->BuildSetting中的Architectures, Valid architectures和Build Active Architecture Only的配置项都是涉及到架构指令集的。下面具体来说明下含义:

### Architectures

这个配置项代表项目可能支持的指令集, 即支持指令集是通过编译生成对应的二进制数据包实现的，如果支持的指令集数目有多个，就会编译出包含多个指令集代码的数据包，造成最终编译的包很大。
		
### Valid architectures

这个配置项代表将要编译的有效指令集, 用来限制可能被支持的指令集的范围，无错误前提下, Valid architectures 和 Architecture两个集合的交集最高指令集才是最终编译生成的版本, 即目标指令集。
举个例子，将Architectures支持arm指令集设置为：`armv7`,`armv7s`，对应的Valid Architectures的支持的指令集设置为：`armv7s`,`arm64`，那么此时，XCode生成二进制包所支持的指令集只有`armv7s`。 
		
### Build Active Architecture Only

这个配置项代表是否只编译当前设备适用的指令集, 如果这个参数设为YES，使用iPhone 6调试，那么最终生成的一个支持`arm64`指令集的Binary。一般在DEBUG模式下设为YES，RELEASE设为NO。


### 注意事项

在上面的文章中说过, 处理器的所有指令集原则上都是向下兼容的, 就像这样 `arm64` > `armv7s` > `armv7` >`armv6`, 所以生成的二进制包将会支持当前支持的指令集以及比它小的指令集。
Architectures选取最高支持的指令集为目标指令集,然后和Valid Architectures取交集的最高指令集才是最终生成二进制包支持的指令集。

举例, 以iPhone 6S为例, 设备匹配指令集是`arm64`。

1. 生成二进制包支持的指令集等于当前设备的CPU指令集
Architectures:  `armv7`, `armv7s`, `arm64`
Valid Architectures:  `armv6`, `armv7s`, `arm64`
生成二进制包的目标指令集： `arm64` 

2. 生成二进制包支持的指令集小于当前设备的CPU指令集
Architectures:  `armv6`, `armv7`
Valid Architectures: `armv6`, `armv7`, `arm64`
生成二进制包的目标指令集： `armv7`

3. 特殊情况`armv6`
Architectures: `armv6`
Valid Architectures: `armv6`, `armv7s`, `arm64`
生成二进制包的目标指令集： 无
虽然编译成功了，但是并没有任何目标生成， 因为从XCode4.5开始，就不再支持`armv6`指令集，所以列表中写了也是白写。

4. 生成二进制包出错
Architectures: `armv7`, `armv7s`, `arm64`
Valid Architectures: `armv7`，`armv7s`
生成二进制包的目标指令集： 编译出错信息
当前设备为iPhone 6S，其默认指令集为`arm64`. 从Architectures中选取的目标指令集应该为`arm64`, 但是目标指令集并不在Valid Architectures中,所以不是有效指令集，则编译将会出错。


-------

欢迎指正, [wangyanchang21](https://github.com/wangyanchang21).


