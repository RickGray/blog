---
layout: post
title: 以太坊智能合约安全入门了解一下（下）
tags: [ethereum, blockchain, security]
---

**（注：本文分上/下两部分完成，上篇链接[《以太坊智能合约安全入门了解一下（上）》](http://rickgray.me/2018/05/17/ethereum-smart-contracts-vulnerabilites-review/)）** 接上篇

#### 3. Arithmetic Issues

算数问题？通常来说，在编程语言里算数问题导致的漏洞最多的就是整数溢出了，整数溢出又分为上溢和下溢。整数溢出的原理其实很简单，这里以 8 位无符整型为例，8 位整型可表示的范围为 `[0, 255]`，`255` 在内存中存储按位存储的形式为（下图左）：

![](/images/articles/2018-05-26-ethereum-smart-contracts-vulnerabilities-review-part2/8.png)

8 位无符整数 255 在内存中占据了 8bit 位置，若再加上 1 整体会因为进位而导致整体翻转为 0，最后导致原有的 8bit 表示的整数变为 0.

如果是 8 位有符整型，其可表示的范围为 `[-128, 127]`，`127` 在内存中存储按位存储的形式为（下图左）：

![](/images/articles/2018-05-26-ethereum-smart-contracts-vulnerabilities-review-part2/9.png)

在这里因为高位作为了符号位，当 `127` 加上 1 时，由于进位符号位变为 `1`（负数），因为符号位已翻转为 `1`，通过还原此负数值，最终得到的 8 位有符整数为 `-128`。

上面两个都是整数上溢的图例，同样整数下溢 `(uint8)0-1=(uint8)255`, `(int8)(-128)-1=(int8)127`。

在 `withdraw(uint)` 函数中首先通过 `require(balances[msg.sender] - _amount > 0)` 来确保账户有足够的余额可以提取，随后通过 `msg.sender.transfer(_amount)` 来提取 Ether，最后更新用户余额信息。这段代码若是一个没有任何安全编码经验的人来审计，代码的逻辑处理流程似乎看不出什么问题，但是如果是编码经验丰富或者说是安全研究人员来看，这里就明显存在整数溢出绕过检查的漏洞。

在 Solidity 中 `uint` 默认为 256 位无符整型，可表示范围 `[0, 2**256-1]`，在上面的示例代码中通过做差的方式来判断余额，如果传入的 `_amount` 大于账户余额，则 `balances[msg.sender] - _amount` 会由于整数下溢而大于 0 绕过了条件判断，最终提取大于用户余额的 Ether，且更新后的余额可能会是一个极其大的数。

```javascript
pragma solidity ^0.4.10;

contract MyToken {
    mapping (address => uint) balances;
        
    function balanceOf(address _user) returns (uint) { return balances[_user]; }
    function deposit() payable { balances[msg.sender] += msg.value; }
    function withdraw(uint _amount) {
        require(balances[msg.sender] - _amount > 0);  // 存在整数溢出
        msg.sender.transfer(_amount);
        balances[msg.sender] -= _amount;
    }
}
```

简单的利用过程演示：

![](/images/articles/2018-05-26-ethereum-smart-contracts-vulnerabilities-review-part2/integer_down_overflow.gif)

为了避免上面代码造成的整数溢出，可以将条件判断改为 `require(balances[msg.sender] > _amount)`，这样就不会执行算术操作进行进行逻辑判断，一定程度上避免了整数溢出的发生。

Solidity 除了简单的算术操作会出现整数溢出外，还有一些需要注意的编码细节，稍不注意就可能形成整数溢出导致无法执行正常代码流程：

- 数组 `length` 为 256 位无符整型，仔细对 `array.length++` 或者 `array.length--` 操作进行溢出校验；
- 常见的循环变量 `for (var i = 0; i < items.length; i++) ...` 中，`i` 为 8 位无符整型，当 `items` 长度大于 256 时，可能造成 `i` 值溢出无法遍历完全；

关于合约整数溢出的漏洞并不少见，可以看看最近曝光的几起整数溢出事件：[《代币变泡沫，以太坊Hexagon溢出漏洞比狗庄还过分》](https://www.anquanke.com/post/id/145520)，[《Solidity合约中的整数安全问题——SMT/BEC合约整数溢出解析》](https://www.anquanke.com/post/id/106382)

**为了防止整数溢出的发生，一方面可以在算术逻辑前后进行验证，另一方面可以直接使用 OpenZeppelin 维护的一套智能合约函数库中的 [SafeMath](https://github.com/OpenZeppelin/openzeppelin-solidity/blob/master/contracts/math/SafeMath.sol) 来处理算术逻辑。**

#### 4. Unchecked Return Values For Low Level Calls

未严格判断不安全函数调用返回值，这类型的漏洞其实很好理解，在前面讲 Reentrancy 实例的时候其实也涉及到了底层调用返回值处理验证的问题。上篇已经总结过几个底层调用函数的返回值和异常处理情况，这里再回顾一下 3 个底层调用 `call()`, `delegatecall()`, `callcode()` 和 3 个转币函数 `call.value()()`, `send()`, `transfer()`：

**- call()**

`call()` 用于 Solidity 进行外部调用，例如调用外部合约函数 `<address>.call(bytes4(keccak("somefunc(params)"), params))`，外部调用 `call()` 返回一个 bool 值来表明外部调用成功与否：

![](/images/articles/2018-05-26-ethereum-smart-contracts-vulnerabilities-review-part2/13.png)

**- delegatecall()**

除了 `delegatecall()` 会将外部代码作直接作用于合约上下文以外，其他与 `call()` 一致，同样也是只能获取一个 bool 值来表示调用成功或者失败（发生异常）。

**- callcode()**

`callcode()` 其实是 `delegatecall()` 之前的一个版本，两者都是将外部代码加载到当前上下文中进行执行，但是在 `msg.sender` 和 `msg.value` 的指向上却有差异。

例如 Alice 通过 `callcode()` 调用了 Bob 合约里同时 `delegatecall()` 了 Wendy 合约中的函数，这么说可能有点抽象，看下面的代码：

![](/images/articles/2018-05-26-ethereum-smart-contracts-vulnerabilities-review-part2/14.png)

如果还是不明白 `callcode()` 与 `delegatecall()` 的区别，可以将上述代码在 remix-ide 里测试一下，观察两种调用方式在 `msg.sender` 和 `msg.value` 上的差异。

**- call.value()()**

在合约中直接发起 TX 的函数之一（相当危险），

**- send()**

通过 `send()` 函数发送 Ether 失败时直接返回 false；这里需要注意的一点就是，`send()` 的目标如果是合约账户，则会尝试调用它的 fallbcak() 函数，fallback() 函数中执行失败，`send()` 同样也只会返回 false。但由于只会提供 2300 Gas 给 fallback() 函数，所以可以防重入漏洞（恶意递归调用）。

**- transfer()**

`transfer()` 也可以发起 Ether 交易，但与 `send()` 不同的时，`transfer()` 是一个较为安全的转币操作，当发送失败时会自动回滚状态，该函数调用没有返回值。同样的，如果 `transfer()` 的目标是合约账户，也会调用合约的 fallback() 函数，并且只会传递 2300 Gas 用于 fallback() 函数执行，可以防止重入漏洞（恶意递归调用）。

这里以一个简单的示例来说明严格验证底层调用返回值的重要性：

```javascript
function withdraw(uint256 _amount) public {
	require(balances[msg.sender] >= _amount);
	balances[msg.sender] -= _amount;
	etherLeft -= _amount;
	msg.sender.send(_amount);  // 未验证 send() 返回值，若 msg.sender 为合约账户 fallback() 调用失败，则 send() 返回 false
}
```

上面给出的提币流程中使用 `send()` 函数进行转账，因为这里没有验证 `send()` 返回值，如果 msg.sender 为合约账户 fallback() 调用失败，则 send() 返回 false，最终导致账户余额减少了，钱却没有拿到。

关于该类问题可以详细了解一下 [King of the Ether](https://www.kingoftheether.com/postmortem.html)。

#### 5. Denial of Service - 拒绝服务

DoS 无处不在，在 Solidity 里也是，与其说是拒绝服务漏洞不如简单的说成是 “不可恢复的恶意操作或者可控制的无限资源消耗”。简单的说就是对以太坊合约进行 DoS 攻击，可能导致 Ether 和 Gas 的大量消耗，更严重的是让原本的合约代码逻辑无法正常运行。

下面一个例子（代码改自 DASP 中例子）：

```javascript
pragma solidity ^0.4.10;

contract PresidentOfCountry {
    address public president;
    uint256 price;

    function PresidentOfCountry(uint256 _price) {
        require(_price > 0);
        price = _price;
        president = msg.sender;
    }

    function becomePresident() payable {
        require(msg.value >= price); // must pay the price to become president
        president.transfer(price);   // we pay the previous president
        president = msg.sender;      // we crown the new president
        price = price * 2;           // we double the price to become president
    }
}
```

一个简单的类似于 KingOfEther 的合约，按合约的正常逻辑任何出价高于合约当前 `price` 的都能成为新的 president，原有合约里的存款会返还给上一人 president，并且这里也使用了 `transfer()` 来进行 Ether 转账，看似没有问题的逻辑，但不要忘了，以太坊中有两类账户类型，如果发起 `becomePresident()` 调用的是个合约账户，并且成功获取了 president，如果其 fallback() 函数恶意进行了类似 `revert()` 这样主动跑出错误的操作，那么其他账户也就无法再正常进行 becomePresident 逻辑成为 president 了。

简单的攻击代码如下：

```javassscript
contract Attack {
    function () { revert(); }
    
    function Attack(address _target) payable {
        _target.call.value(msg.value)(bytes4(keccak256("becomePresident()")));
    }
}
```

使用 remix-ide 模拟攻击流程：

![](/images/articles/2018-05-26-ethereum-smart-contracts-vulnerabilities-review-part2/president_of_country_demo.gif)

#### 6. Bad Randomness - 可预测的随机处理

伪随机问题一直都存在于现代计算机系统中，但是在开放的区块链中，像在以太坊智能合约中编写的基于随机数的处理逻辑感觉就有点不切实际了，由于人人都能访问链上数据，合约中的存储数据都能在链上查询分析得到。如果合约代码没有严格考虑到链上数据公开的问题去使用随机数，可能会被攻击者恶意利用来进行 “作弊”。

摘自 DASP 的代码块：

```javascript
uint256 private seed;
    
function play() public payable {
	require(msg.value >= 1 ether);
	iteration++;
	uint randomNumber = uint(keccak256(seed + iteration));
	if (randomNumber % 2 == 0) {
		msg.sender.transfer(this.balance);
	}
}
```

这里 `seed` 变量被标记为了私有变量，前面有说过链上的数据都是公开的，`seed` 的值可以通过扫描与该合约相关的 TX 来获得。获取 `seed` 值后，同样的 `iteration` 值也是可以得到的，那么整个 `uint(keccak256(seed + iteration))` 的值就是可预测的了。

就 DASP 里面提到的，还有一些合约喜欢用 `block.blockhash(uint blockNumber) returns (bytes32)` 来获取一个随机哈希，但是这里切记不能使用 `block.number` 也就是当前块号来作为 `blockNumber` 的值，因为在官方文档中明确写了：

> block.blockhash(uint blockNumber) returns (bytes32): hash of the given block - only works for 256 most recent blocks excluding current

意思是说 `block.blockhash()` 只能使用近 256 个块的块号来获取 Hash 值，并且还强调了不包含当前块，如果使用当前块进行计算 `block.blockhash(block.numbber)` 其结果始终为 `0x0000000.....`：

![](/images/articles/2018-05-26-ethereum-smart-contracts-vulnerabilities-review-part2/15.png)

同样的也不能使用 `block.timestamp`, `now` 这些可以由矿工控制的值来获取随机数。

一切链上的数据都是公开的，想要获取一个靠谱的随机数，使用链上的数据看来是比较难做到的了，这里有一个独立的项目 [Oraclize](https://github.com/oraclize/ethereum-api) 被设计来让 Smart Contract 与互联网进行交互，有兴趣的同学可以深入了解一下。（附上基于 Oraclize 的随机数获取方法 [randomExample](https://github.com/oraclize/ethereum-examples/blob/master/solidity/random-datasource/randomExample.sol)）

#### 7. Front Running - 提前交易

“提前交易”，其实在学习以太坊智能合约漏洞之前，我还并不知道这类漏洞类型或者说是攻击手法（毕竟我对金融一窍不通）。简单来说，“提前交易”就是某人提前获取到交易者的具体交易信息（或者相关信息），抢在交易者完成操作之前，通过一系列手段（通常是提高报价）来抢在交易者前面完成交易。

![](/images/articles/2018-05-26-ethereum-smart-contracts-vulnerabilities-review-part2/16.png)

在以太坊中所有的 TX 都需要经过确认才能完全记录到链上，而每一笔 TX 都需要带有相关手续费，而手续费的多少也决定了该笔 TX 被矿工确认的优先级，手续费高的 TX 会被优先得到确认，而每一笔待确认的 TX 在广播到网络之后就可以查看具体的交易详情，一些涉及到合约调用的详细方法和参数可以被直接获取到。那么这里显然就有 Front-Running 的隐患存在了，示例代码就不举了，直接上图（形象一点）：

![](/images/articles/2018-05-26-ethereum-smart-contracts-vulnerabilities-review-part2/17.png)

在 [etherscan.io](https://etherscan.io/txsPending) 就能看到还未被确认的 TX，并且能给查看相关数据：

![](/images/articles/2018-05-26-ethereum-smart-contracts-vulnerabilities-review-part2/18.png)

**（当然了，为了防止信息明文存储在 TX 中，可以对数据进行加密和签名）**

#### 8. Time Manipulation

“时间篡改”（DASP 给的名字真抽象 XD），说白了一切与时间相关的漏洞都可以归为 "Time Manipulation"。在 Solidity 中，`block.timestamp` （别名 `now`）是受到矿工确认控制的，也就是说一些合约依赖于 `block.timestamp` 是有被攻击利用的风险的，当攻击者有机会作为矿工对 TX 进行确认时，由于 `block.timestamp` 可以控制，一些依赖于此的合约代码即预知结果，攻击者可以选择一个合适的值来到达目的。（当然了 `block.timestamp` 的值通常有一定的取值范围，出块间隔有规定 XD）

该类型我还没有找到一个比较好的例子，所以这里就不给代码演示了。:)

#### 9. Short Address Attack - 短地址攻击

在我着手测试和复现合约漏洞类型时，短地址攻击我始终没有在 remix-ide 上测试成功（道理我都懂，咋就不成功呢？）。虽然漏洞没有复现，但是漏洞原理我还是看明白了，下面就详细地说明一下短地址攻击的漏洞原理吧。

首先我们以外部调用 `call()` 为例，外部调用中 `msg.data` 的情况：

![](/images/articles/2018-05-26-ethereum-smart-contracts-vulnerabilities-review-part2/19.png)

在 remix-ide 中部署此合约并调用 `callFunc()` 时，可以得到日志输出的 `msg.data` 值：

```
0x4142c000000000000000000000000000000000000000000000000000000000000000001e
```

其中 `0x4142c000` 为外部调用的函数名签名头 4 个字节（`bytes4(keccak256("foo(uint32,bool)"))`），而后面 32 字节即为传递的参数值，`msg.data` 一共为 4 字节函数签名加上 32 字节参数值，总共 `4+32` 字节。

看如下合约代码：

```javascript
pragma solidity ^0.4.10;

contract ICoin {
    address owner;
    mapping (address => uint256) public balances;

    modifier OwnerOnly() { require(msg.sender == owner); _; }
    
    function ICoin() { owner = msg.sender; }
    function approve(address _to, uint256 _amount) OwnerOnly { balances[_to] += _amount; }
    function transfer(address _to, uint256 _amount) {
        require(balances[msg.sender] > _amount);
        balances[msg.sender] -= _amount;
        balances[_to] += _amount;
    }
}
```

具体代币功能的合约 ICoin，当 A 账户向 B 账户转代币时调用 `transfer()` 函数，例如 A 账户（0x14723a09acff6d2a60dcdf7aa4aff308fddc160c）向 B 账户（0x4b0897b0513fdc7c541b6d9d7e929c4e5364d2db）转 8 个 ICoin，`msg.data` 数据为：

```
0xa9059cbb  -> bytes4(keccak256("transfer(address,uint256)")) 函数签名
0000000000000000000000004b0897b0513fdc7c541b6d9d7e929c4e5364d2db  -> B 账户地址（前补 0 补齐 32 字节）
0000000000000000000000000000000000000000000000000000000000000008  -> 0x8（前补 0 补齐 32 字节）
```

那么短地址攻击是怎么做的呢，攻击者找到一个末尾是 `00` 账户地址，假设为 `0x4b0897b0513fdc7c541b6d9d7e929c4e5364d200`，那么正常情况下整个调用的 `msg.data` 应该为：

```
0xa9059cbb  -> bytes4(keccak256("transfer(address,uint256)")) 函数签名
0000000000000000000000004b0897b0513fdc7c541b6d9d7e929c4e5364d200  -> B 账户地址（注意末尾 00）
0000000000000000000000000000000000000000000000000000000000000008  -> 0x8（前补 0 补齐 32 字节）
```

但是如果我们将 B 地址的 `00` 吃掉，不进行传递，也就是说我们少传递 1 个字节变成 `4+31+32`：

```
0xa9059cbb  -> bytes4(keccak256("transfer(address,uint256)")) 函数签名
0000000000000000000000004b0897b0513fdc7c541b6d9d7e929c4e5364d2  -> B 地址（31 字节）
0000000000000000000000000000000000000000000000000000000000000008  -> 0x8（前补 0 补齐 32 字节）
```

当上面数据进入 EVM 进行处理时，会犹豫参数对齐的问题后补 `00` 变为：

```
0xa9059cbb
0000000000000000000000004b0897b0513fdc7c541b6d9d7e929c4e5364d200
0000000000000000000000000000000000000000000000000000000000000800
```

也就是说，恶意构造的 `msg.data` 通过 EVM 解析补 0 操作，导致原本 `0x8 = 8` 变为了 `0x800 = 2048`。

上述 EVM 对畸形字节的 `msg.data` 进行补位操作的行为其实就是短地址攻击的原理（但这里我真的没有复现成功，希望有成功的同学联系我一起交流）。

短地址攻击通常发生在接受畸形地址的地方，如交易所提币、钱包转账，所以除了在编写合约的时候需要严格验证输入数据的正确性，而且在 Off-Chain 的业务功能上也要对用户所输入的地址格式进行验证，防止短地址攻击的发生。

同时，老外有一篇介绍 [Analyzing the ERC20 Short Address Attack](https://ericrafaloff.com/analyzing-the-erc20-short-address-attack/) 原理的文章我觉得非常值得学习。

#### - Unknown Unknowns - 其他未知，:) 未知漏洞，没啥好讲的，为了跟 DASP 保持一致而已

### III. 自我思考

前后花了 2 周多的时间去看以太坊智能合约相关知识以及本文（上/下）的完成，久违的从 0 到 1 的感觉又回来了。多的不说了，我应该也算是以太坊智能合约安全入门了吧，近期出的一些合约漏洞事件也在跟，分析和复现也是完全 OK 的，漏洞研究原理不变，变得只是方向而已。期待同更多的区块链安全研究者交流和学习。

#### 1. 以太坊中合约账户的私钥在哪？可以不通过合约账户代码直接操作合约账户中的 Ether 吗？

StackExchange 上有相关问题的回答 [“Where is the private key for a contract stored?”](https://ethereum.stackexchange.com/questions/185/where-is-the-private-key-for-a-contract-stored)，但是我最终也没有看到比较官方的答案。但可以知道的就是，合约账户是由部署时的合约代码控制的，**不确定是否有私钥可以直接控制合约进行 Ether 相关操作**（讲道理应该是不行的）。

#### 2. 使用 keccak256() 进行函数签名时的坑？- 参数默认位数标注

在使用 keccak256 对带参函数进行签名时，需要注意要严格制定参数类型的位数，如：

```javascript
function somefunc(uint n) { ... }
```

对上面函数进行签名时，定义时参数类型为 `uint`，而 `uint` 默认为 256 位，也就是 `uint256`，所以在签名时应该为 `keccak256("somefunc(uint256)")`，千万不能写成 `keccak256("somefunc(uint)")`。

### 参考链接：

- [http://solidity.readthedocs.io/en/v0.4.21/units-and-global-variables.html#special-variables-and-functions](http://solidity.readthedocs.io/en/v0.4.21/units-and-global-variables.html#special-variables-and-functions)
- [https://github.com/oraclize/ethereum-api](https://github.com/oraclize/ethereum-api)
- [https://ericrafaloff.com/analyzing-the-erc20-short-address-attack/](https://ericrafaloff.com/analyzing-the-erc20-short-address-attack/)