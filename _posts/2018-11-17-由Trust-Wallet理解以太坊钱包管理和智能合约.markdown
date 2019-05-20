---
title:  "由Trust Wallet理解以太坊钱包管理和智能合约"
date:   2018-11-17 23:07:17
categories: [区块链]
tags: [区块链]
---

[以太坊钱包 Trust项目解读之架构和流程](https://wangyanchang21.github.io/2018/%E7%94%B1Trust-Wallet%E7%90%86%E8%A7%A3%E4%BB%A5%E5%A4%AA%E5%9D%8A%E9%92%B1%E5%8C%85%E7%AE%A1%E7%90%86%E5%92%8C%E6%99%BA%E8%83%BD%E5%90%88%E7%BA%A6)

[由Trust Wallet理解以太坊钱包管理和智能合约](https://blog.csdn.net/wangyanchang21/article/details/83862016)

------

在前一篇文章中, 已经介绍过`Trust`的项目架构、业务流程等了。这篇文章将会解读一些核心的功能, 包括前一篇文章提到的`EtherKeystore`这个业务类, 还有网络层的如何调用智能合约、其它调用合约的方式, 以及以太坊交易的结构和流程等。

## 钱包管理

钱包管理就要提到一个类`EtherKeystore`, 应用的核心业务的处理类, 有钱包管理(创建、删除、导入、导出)、助记词转化、签名工作、私钥管理等功能。
`EtherKeystore`中使用了由`Trust`开源的了两个库: [TrustKeystore](https://github.com/TrustWallet/trust-keystore): 用于管理钱包的通用以太坊密钥库。[TrustCore](https://github.com/TrustWallet/trust-core): 区块链核心的数据结构和算法。还有[CryptoSwift](https://github.com/krzyzanowskim/CryptoSwift), 一个标准的安全加密算法集合的库。

### 钱包创建

在`EtherKeystore`类中, 封装了钱包的创建, 主要使用了`TrustKeystore`库、`TrustCore`库中关于公私钥和地址的API、以及密码学的库`CryptoSwift`。我下面所说的整个流程也包括这些库中的源码逻辑, 先创建密钥对(或者助记词), 再利用本地生成的随机密码对密钥进行加密保存, 然后生成钱包, 将钱包、获取私钥的密码以及`KeystoreKey`保存到本地。

`Trust`默认的方式是生成助记词, 这种方式其实是私钥的一种管理方式, 助记词是由私钥通过某种算法派生出来的, `TrustCore`中的`Crypto`就是这个功能。而且当你用到私钥的时候, 你还可以把你的助记词通过对应的算法在转译成私钥。所以它只是一种私钥的存储方式, 下面文章中以私钥为例来讲述整个流程。

#### 创建公钥私钥

创建钱包就相当于生成一对密钥, 公钥(PublicKey)和私钥(PrivateKey)。公钥其实就相当于你账户在区块链中的地址(Address); 私钥就相当于你钱包的账号密码, 它是证明你是钱包主人的唯一证明, 一旦丢失就不可找回。当然, 公钥并不完全等于地址, 地址是由公钥经过一系列的算法生成的, 需要经过`SHA3-256`(Keccak)哈希然后转化为符合`EIP55`规则的字符串。
```
 (sk, pk) = generateKeys(keysize) 
 ```
上面这段伪代码中, generateKeys方法把 keysize作为输入, 来产生一对公钥和私钥。私钥sk被安全保存## ，并用来签名一段消息；公钥pk是人人都可以找到的，拿到它，就可以用来验证你的签名。下图是`TrustCore`中对以太坊私钥和地址的`keysize`定义, 私钥是32字节, 公钥地址是20字节, 所以十六进制的私钥长度为64位, 而公钥地址长度为40位。

<center>
	<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/trustwallet-2/20181107182953343.png" width="70%" img/>
</center>


具体来说, 创建公钥和私钥的功能是由`TrustCore`中的`PrivateKey`来完成的。而且是通过苹果官方的`Security`库来创建的公钥和私钥, 经过整理密钥对生成和获取过程如下:

``` swift
func getPrivatePublicKey() -> (String, String) {
    
    let privateAttributes: [String: Any] = [
        kSecAttrIsExtractable as String: true,
        ]
    let parameters: [String: Any] = [
        kSecAttrKeyType as String: kSecAttrKeyTypeEC,
        kSecAttrKeySizeInBits as String: 256,
        kSecPrivateKeyAttrs as String: privateAttributes,
        ]
    
    // PrivateKey To String
    guard let privateKey = SecKeyCreateRandomKey(parameters as CFDictionary, nil) else {
        fatalError("Failed to generate key pair")
    }
    guard var priRepresentation = SecKeyCopyExternalRepresentation(privateKey, nil) as Data? else {
        fatalError("Failed to extract new private key")
    }
    defer {
        priRepresentation.replaceSubrange(0 ..< priRepresentation.count, with: repeatElement(0, count: priRepresentation.count))
    }
    
    let priData = Data(priRepresentation.suffix(32))
    var priString = ""
    for byte in priData {
        priString.append(String(format: "%02x", byte))
    }
    
    
    // PublicKey To String
    guard let publicKey = SecKeyCopyPublicKey(privateKey) else {
        fatalError("Failed to get publickey")
    }
    guard var pubRepresentation = SecKeyCopyExternalRepresentation(publicKey, nil) as Data? else {
        fatalError("Failed to extract new public key")
    }
    defer {
        pubRepresentation.replaceSubrange(0 ..< pubRepresentation.count, with: repeatElement(0, count: pubRepresentation.count))
    }
    
    let pubData = Data(pubRepresentation.suffix(32))
    var pubString = ""
    for byte in pubData {
        pubString.append(String(format: "%02x", byte))
    }
    
    return (priString, pubString)
}
```

#### 使用随机密码对私钥加密

在生成了私钥之后, 将在`KeystoreKeyHeader`类中, 这里使用了`CryptoSwift`(安全加密算法集合的库)对私钥进行加密。使用`AES-128`算法进行对称加密后, 将这些数据以`KeystoreKeyHeader`类型保存在`KeystoreKey`中。

#### 创建 Wallet

在前两个步骤的基础之上, 就可以创建一个`Wallet`了, 并将`Wallet`加入到当前的账户中。也会计算或者获取一些参数存储在`Wallet`中, 如公钥地址Address, Account、`KeystoreKey`等。

#### 保存到本地

`KeyStore`会将当前钱包账户的`KeystoreKey`数据存储在本地文件中。文件以"UTC+时间戳+钱包唯一标识"为名称存储在本地, 其中存储的是上面`KeystoreKey`的数据。这些数据用户每次启动时, 将会由这些数据再次生成所有的`Wallet`数据。
当然, 私钥当然也是需要保存的, [前一篇文章](https://wangyanchang21.github.io/2018/%E7%94%B1Trust-Wallet%E7%90%86%E8%A7%A3%E4%BB%A5%E5%A4%AA%E5%9D%8A%E9%92%B1%E5%8C%85%E7%AE%A1%E7%90%86%E5%92%8C%E6%99%BA%E8%83%BD%E5%90%88%E7%BA%A6)中说过了, 这样的敏感信息保存在`keychain`中。但`keychain`并不是直接存储这私钥, 而是将获取私钥的密码保存在其中了。以钱包的id为key值, 将获取私钥的密码保存子`keychain`之中, 拿出密码后, 再使用`KeystoreKey`进行`AES-128`对称解密, 获取私钥, 便可以使用了。所以, `KeystoreKey`这个类的主要功能是对私钥和助记词的管理以及对私钥的加解密。

另外, 这样拥有`PrivateKey`的钱包账户是不需要存储在`Realm`数据库中的。只有一种需要保存到本地的`Realm`数据库中, 那就是导入地址钱包, 下面将会说明。


### 钱包的导入

钱包导入相对于钱包的创建来说, 只是不需要自己去生成公钥和私钥对了, 剩下的流程还是一样的。当然导入时会有三种方式, 除了之前提到的私钥和助记词的方式, 还有地址的方式。
钱包地址是公开的, 当然你也可以导入, 也可以查看这个钱包的任何数据, 但因为你不具备它的私钥, 所以你不可以进行签名或者说任何写入区块链的操作。所以这种方式, 就不需要`KeyStore`进行操作了, 只需要`EtherKeystore`进行本地操作, 将其放入本地的`Realm`数据库中, 那就是导入地址钱包。当启动应用时, 将会以两者组成的数据为本地钱包列表。

### 钱包导出、删除等

钱包导出, 当然也会分三种方式, 私钥和助记词的方式, 还有地址的方式。在`keychain`中将密码取出, 然后通过`KeystoreKey`解密到私钥或者助记词, 导出。地址的方式, 就是直接导出地址。

如果你把上面的钱包创建条理理清楚了, 你就可以想到删除只是钱包创建的逆过程, 但没有那么复杂。只需要验证你的私钥是正确的就可以将你本地`KeystoreKey`删除了。

### EtherKeystore 模块结构图

下图中画了 `EtherKeystore`在创建或者导入钱包时的流程, 可一清楚的看到这个模块的结构。绿色的部分是`TrustCore`和`TrustKeystore`库中的调用, 浅蓝色是数据层的一些处理。

<center>
	<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/trustwallet-2/20181109110331147.png" width="70%" img/>
</center>

## 智能合约

在[前一篇文章](https://wangyanchang21.github.io/2018/%E7%94%B1Trust-Wallet%E7%90%86%E8%A7%A3%E4%BB%A5%E5%A4%AA%E5%9D%8A%E9%92%B1%E5%8C%85%E7%AE%A1%E7%90%86%E5%92%8C%E6%99%BA%E8%83%BD%E5%90%88%E7%BA%A6)中的网络层中, 对只能合约以及具体网络层业务逻辑没有做详细说明。这里将会讨论几个问题, 网络层具体方案, 以太坊智能合约的调用。

### 合约调用方式

在以太坊的官方文档中提供了两种 API, 一个种是[JSON RPC API](https://github.com/ethereum/wiki/wiki/JSON-RPC), 一种是[JavaScript API](https://github.com/ethereum/wiki/wiki/JavaScript-API)。

#### JavaScript API

虽然看起来是两种 API, 其实后者是通过RPC调用与本地节点进行通信的。也就可以理解为 `JavaScript API`是对 `JSON RPC API`的封装, 方便了从JavaScript应用程序内部与`ethereum节点`通信。官方开源的库[web3.js](https://github.com/ethereum/web3.js)就是做了这个事情。

#### JSON RPC API

[JSON-RPC](https://www.jsonrpc.org/specification)是一种轻量级的远程过程调用(RPC)协议。该规范主要定义了一些数据结构和处理的规则。它与传输无关, 因为这些概念可以通过`Socket`、`HTTP`, 或者其它的消息传递环境中使用。它使用 JSON([RFC 4627](http://www.ietf.org/rfc/rfc4627.txt))作为数据格式。

默认的JSON-RPC端点：
| Client | URL |
| --- | --- |
| C++ | http://localhost:8545 |
| Go | http://localhost:8545 |
| Py | http://localhost:4000 |
| Parity |	http://localhost:8545 |

RPC的支持:情况 
|   | cpp-ethereum | go-ethereum | py-ethereum | parity |
| --- | --- | --- | --- | --- |
| JSON-RPC 1.0 | ✓ |  |   |   | 
| JSON-RPC 2.0 | ✓ | ✓ | ✓ | ✓ |
| Batch requests | ✓ | ✓ | ✓ | ✓ |
| HTTP | ✓ | ✓ | ✓ | ✓ |
| IPC | ✓ | ✓ |   | ✓ |
| WS |   | ✓ |   | ✓ | 


### 合约调用

当然, 在`Trust`的 iOS端是通过 `JSON RPC Over HTTP`的方式进行智能合约调用的。项目中针对合约调用的请求, 网络层的设计是 [APIKit](https://github.com/ishkawa/APIKit) + [JSONRPCKit](https://github.com/bricklife/JSONRPCKit) 的方式。

#### JSON RPC Over HTTP

在项目中, 以太坊智能合约调用都是`JSON RPC Over HTTP`的方式, 而且所使用的以太坊节点[前一篇文章](https://wangyanchang21.github.io/2018/%E7%94%B1Trust-Wallet%E7%90%86%E8%A7%A3%E4%BB%A5%E5%A4%AA%E5%9D%8A%E9%92%B1%E5%8C%85%E7%AE%A1%E7%90%86%E5%92%8C%E6%99%BA%E8%83%BD%E5%90%88%E7%BA%A6)网络层中就提到过。
``` swift
var remoteURL: URL {
        let urlString: String = {
            switch self {
            case .main: return "https://api.trustwalletapp.com"
            case .classic: return "https://classic.trustwalletapp.com"
            case .callisto: return "https://callisto.trustwalletapp.com"
            case .poa: return "https://poa.trustwalletapp.com"
            case .gochain: return "https://gochain.trustwalletapp.com"
            }
        }()
        return URL(string: urlString)!
    }
```
网络层结构应该如下图所示:

<center>
	<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/trustwallet-2/20181109180054191.png" width="40%" img/>
</center>


当你明白这种网络结构后, 在来看`Trust`中, 统一使用`xxxRequest`的命名来封装`JSONRPCKit`的应用组件。其中定义了`method`、`parameters`、response的转化等, 这里的`method`就是调用以太坊智能合约的接口名称。项目中, 统一使用`xxxProvider`的命名, 按功能对`APIKit`的请求组件进行封装。当然, 没有这层抽象的HTTP请求也是可以的。

下面图片中, `Trust`中涉及到一些 API: `eth_estimateGas`、`eth_sendRawTransaction`、`eth_gasPrice`、`eth_blockNumber`、`eth_getTransactionByHash`、`eth_call`、`eth_getBalance`。下面详细列出了项目中合约调用的类和具体使用的以太坊 API, 它们是一一对应的关系。

<center>
	<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/trustwallet-2/2018110917531565.png" width="70%" img/>
</center>



#### Web3.swift

`Trust`项目中并没有使用`web3`的方式进行合约调用, 但是我还是想说一说这种方式。这是因为除了以太坊官方的对 JavaScript API的`web3`库以外, 还有一个纯Swift写的库[Web3.swift](https://github.com/Boilertalk/Web3.swift)。它是可以用于在以太坊网络中签署交易并与智能合约进行交互, 而且可以直接使用于你的iOS客户端。假如你的网络层用`Web3.swift`替换`APIKit` + `JSONRPCKit`这样的话, 将会降低网络层结构复杂度, 且代码简洁性也提高了。

### 网络层其他请求

在`Trust`中, 获取区块链上的数据, 其实分为两种, 一种是上面提到的直接通过智能合约获取的数据。另一种就是`Trust`官网已经封装过的一些接口, 它们是关于多币种的, 大多需要在区块链中去查找, 接口不单一且有大工作量的请求, 如transactions, getTokes等。这些接口是直接使用网络库`Moya`进行封装的, 而没有调用智能合约。而这些`HTTP`请求的服务器是: 
```
let trustAPI = URL(string: "https://public.trustwalletapp.com")
```

`TrustAPI`类中将这些接口清楚的列举了出来, 并且将它们集体封装在`TrustNetwork`类中来管理。

<center>
	<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/trustwallet-2/20181109182907445.png" width="70%" img/>
</center>


到这里, 就将[前一篇文章](https://wangyanchang21.github.io/2018/%E7%94%B1Trust-Wallet%E7%90%86%E8%A7%A3%E4%BB%A5%E5%A4%AA%E5%9D%8A%E9%92%B1%E5%8C%85%E7%AE%A1%E7%90%86%E5%92%8C%E6%99%BA%E8%83%BD%E5%90%88%E7%BA%A6)所遗留的网络层的详情补充完整了。


## 交易

交易, 即`Transaction`, 我这里是指转账交易。上面简单介绍过以太坊上的交易, 并了解交易的 API是 `eth_sendRawTransaction`。下面介绍下在项目中, 转账交易的结构, 以及转账交易在`原生App`和`DApp`中分别是怎样的流程。

### 交易的结构

在项目的主目录中, 有一个`Transfer`模块, 这个模块主要功能就是处理转账交易。在形成一个交易前, 将以定义的`Transfer`类为基础, 封装出一个`Transaction`的结构, 这个结构中包含着发送地址、接收地址、币的数量、交易费等等所有交易相关的数据。最后定义`TransactionConfigurator`类, 对交易进行最外层的业务管理和校验。在`TransactionConfigurator`中经过校验、签名之后的交易才会发送给以太坊节点, 并在矿工挖到矿并将此交易放入区块中, 当前`Token`的转账才算完成。

#### Transfer

`Transfer`中主要包含当前转账发起方的`Token`相关的一些数据, 如地址、合约等等。而且它有类型之分, 及`TransferType`的三种类型, 分别是`Coin`、`ERC20`、`Dapp`, 前两种是原生App的方式, 后一种是浏览器中 DApp的方式。

#### UnconfirmedTransaction

`UnconfirmedTransaction`中, 主要包含当前`Token`的一些信息, 即`Transfer`。还有一个转账接收方的信息, 如地址、币的数量、交易费、Data等等。

#### TransactionConfigurator

`TransactionConfigurator`类, 对交易进行最外层的业务管理和校验。它其中包含全量的`UnconfirmedTransaction`数据, 且还有校验余额是否有效、交易费、交易限制等功能, 最终生成一个经过校验后的完整`SignTransaction`。

#### DappAction

`DappAction`只在`DApp`进行转账交易时, 才能使用到的类。而上面的三个无论是`原生App`还是`DApp`都需要使用到的交易结构类。`DappAction`会将浏览器中传入的消息进行解析, 得到`Method`以及其它数据, 并封装在`DappCommand`里面。然后以浏览器的web标题和URL生成的`DAppRequester`等元素生成`Transfer`。最终这两者, 共同生成的`DappAction`来决定需要进行哪种操作、需要调用合约中的哪种API、还有交易的一些数据等。

#### 交易结构图


<center>
	<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/trustwallet-2/20181114104214461.png" width="70%" img/>
</center>


### 交易的流程

交易流程自然也是分成两, 一种是`原生App`中发起的交易, 一种是`DApp`在浏览器中发起的交易。之前提及的交易结构会在流程中以数据的形式作为重要的参与部分, 这里主要说明交易从发起至交易完成的主要流程, 以及需要调用哪些以太坊智能合约的 API。

#### 原生App发起的交易

**交易发起。** 在`原生App`的钱包首页有着当前账户下的`Token`列表, 而发起的转账交易是在某个具体`Token`中操作的。所以当前的`Transfer`是已经具备的, 而具体的交易接收地址、币的数量以及gas费就需要用户在`SendCoordinator`的模块中自行输入了。

**构建交易数据。** 交易发起后, 我们就具备了构建`UnconfirmedTransaction`和`TransactionConfigurator`的所有数据了。它们的具体情况, 前面已经说明过了, 就不赘述了。构建完成`TransactionConfigurator`后, 进入流程中的`ConfirmCoordinator`模块, 它的功能是让用户来确认交易详情, 以及核实当前`Token`的余额是否足够等。

**智能合约调用。** 当用户确认且余额足够支持转账的情况下, 就需要`SendTransactionCoordinator`来进行核心的转账交易业务, 所以它是一个纯业务的功能类, 并无页面。这时候要根据在`TransactionConfigurator`经过校验的 `Transaction`, 判断其`noce`是否大于0。如果不大于0, 则需要通过`JSON RPC Over HTTP`的方式调用以太坊智能合约的API, 即`eth_getTransactionByHash`对`nonce`进行更新, 然后重新进行判断; 如果大于0, 则`EtherKeystore`对交易进行签名, 然后通过`JSON RPC Over HTTP`的方式调用以太坊智能合约的API, 即`eth_sendRawTransaction`。

**交易回调处理。** 交易结果产生后, 要回调至发起的模块, 还要处理后续的业务。如果交易成功, 会将交易保存到本地的`Realm`数据库等; 如果交易失败, 提示用户交易失败。

到此, 转账交易的流程的闭环完成。在后面的图中也对整个交易流程做了一个梳理。

#### DApp发起的交易

`Trust`具有一个功能齐全的`Web3`浏览器，可与任何分布式的应用程式(DApp)配合使用。这个情景就是当转账交易发生在`DApp`中发起的情况。交易整体的流程与`原生App`中基本一致, 且交易的核心数据结构一致。它们的区别在于发起方式、回调处理, 以及`DApp`中要多一些解析的过程。

**交易发起。** 在`Web3`浏览器中的`DApp`中, 发起转账交易, 发起方式就是`JS`调用`iOS原生`。通过传入的数据, 在`BrowserCoordinator`模块中, 将数据进行解析。

**解析。** 通过`DAppAction`、`DappCommand`、`DAppRequester`等类进行解析, 完成后, 封装入`DAppAction`内, 来决定需要进行哪种操作、需要调用合约中的哪种API。它有6种响应事件, 分别是: 
>1.signMessage

>2.signPersonalMessage

>3.signTypedMessage

>4.signTransaction

>5.sendTransaction

>6.unknown



**构建交易数据** 和 **智能合约调用。** 这两个步骤和原生的之间基本一致, 都是通过数据构建出`Transaction`, 用来做交易准备。然后进行校验, 再调用智能合约。所以就不具体说明了, 请参照`原生App`。

**交易回调处理。** 交易结果产生后, 也要回调至发起的模块, 来处理后续的业务。这里与`原生App`区别是, 除了需要完成`原生App`在成功或失败下完成的流程外, 还需要将交易结果再通知到`Web`, 这样才能形成完整的闭环。所以, 无论回调结果如何, 都会通过`iOS原生`调用`JS`的方式通知`Web`交易的具体情况。

#### 交易流程图

<center>
	<img src="https://raw.githubusercontent.com/wangyanchang21/wangyanchang21.github.io/master/resource/trustwallet-2/20181114151958473.png" width="80%" img/>
</center>


`Trust`项目到这里基本就很清晰了, 这两篇文章虽然只是对`Trust 
wallet`的解读, 很局限。但是由它们能延伸到的知识, 如以太坊的智能合约的知识、钱包和私钥管理的知识等等, 还有你对区块链的认知, 这些不是狭义的。所以无论你认为区块链是好是坏, 或者有没有实际的应用和市场的欢迎, 这门技术都带来了无限创新。










