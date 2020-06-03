# BSV智能合约（二）：计数器合约代码分析

为了实现上一篇文章中说的“计数器合约”，需要介绍BSV上的高级语言[sCrypt](https://scryptdoc.readthedocs.io/en/latest/)，并解释如何用sCrypt实现这个合约。

sCrypt和BSV脚本的关系就像C语言和汇编语言的关系。通过[sCrypt VS Code插件](https://marketplace.visualstudio.com/items?itemName=bsv-scrypt.sCrypt)，可以把sCrypt编译成BSV脚本语言，还可以在VSCode中进行调试。

sCrypt语言实现的[“计数器合约”代码](https://github.com/scrypt-sv/boilerplate/blob/master/tests/testnet/counter.js)如下。一会我们会详细分析。

```c++
contract Counter {
    public function increment(bytes sighashPreimage, int amount) {
        Tx tx = new Tx();
        require(tx.validate(sighashPreimage));
        int len = length(sighashPreimage);
        bytes hashOutputs = sighashPreimage[len - 40 : len - 8];
        bytes scriptCode = Util.readVarint(sighashPreimage[104:]);
        int scriptLen = length(scriptCode);
        int counter = unpack(scriptCode[scriptLen - 1 :]);
        bytes scriptCode_ = scriptCode[: scriptLen - 1] ++ num2bin(counter + 1, 1);
        Sha256 hashOutputs_ = hash256(num2bin(amount, 8) ++ Util.writeVarint(scriptCode_));
        require(hashOutputs == hashOutputs_);
    }
}
```



## 合约原理

关键问题：旧TX的合约output被花费时要检查新TX的output是否符合规则，但旧TX并不知道新TX的output，那怎么检查呢？

解决方案倒也不复杂，在花费旧TX output时，把新TX output作为解锁参数告诉旧TX的output脚本，让脚本对该参数做分析和检查，如果符合条件才允许花费。



## 代码分析

`increment`函数的内容就是“计数器合约”output中的脚本内容，编译后可以分两部分：

1. 逻辑部分：检查新TX output是否符合规则
2. 数据部分：计数器的值。位于output脚本的最后，占1字节。

![output script structure](https://bico.media/7a8dca9ec04f1b6f2a078ab270094fe9479413679fd77e1d9457c76b4874b859)



逻辑部分的主要检查两个条件：

1. 新TX output脚本的逻辑部分与旧TX output脚本的逻辑部分一样。

   满足了新TX的output只能是“计数器”合约，不能是其他类型（如P2PKH）output的需求。

2. 新TX output脚本的数据部分比旧TX output脚本的数据部分的值大1。

   满足了每次执行（花费）合约，计数器值加1且只加1的需求。



### 解锁参数

首先，分析参数声明。

```c++
public function increment(bytes sighashPreimage, int amount)
```

这一行的关键是两个输入参数，这两个输入参数就是花费（执行）合约output时提供的解锁参数，包含在了新TX的input中。`amount`参数不关键，先忽略。重点看`sighashPreimage`参数。这个参数实际上是一系列数据的集合，其中包括了两个重要数据：

1. 新TX output内容的hash值
2. 旧TX output脚本内容

![input parameters](https://bico.media/82d0be05e89b82c8c5c13fb6790f954e28d47a05b790f5bad90be5cf96dd1264)



### 检查sighashPreimage参数真实性

合约的原理很简单，**关键是如何保证解锁参数的真实性**。因为解锁参数在构造新TX时是可以随意输入的，那么如何保证`sighashPreimage`参数中的两个重要数据是真实的（一定与`新TX实际的output数据hash`、`旧TX实际的output脚本`完全相同），而不是随意输入的假数据呢？下面这两行代码就是做这个检查的：

```c++
Tx tx = new Tx();
require(tx.validate(sighashPreimage));
```

你一定会好奇这两行代码到底是如何保证数据真实的，这是合约最精妙的地方，我会在下一篇文章里专门解释。

总之，如果数据不真实，那么脚本就会返回失败，UTXO不能被花费，也就是说合约不能被执行。



### 解析sighashPreimage参数

```c++
int len = length(sighashPreimage);
bytes hashOutputs = sighashPreimage[len - 40 : len - 8];
bytes scriptCode = Util.readVarint(sighashPreimage[104:]);
int scriptLen = length(scriptCode);
int counter = unpack(scriptCode[scriptLen - 1 :]);
```

`sighashPreimage`的结构是固定的，按固定结构从中解析出：新TX output hash值`hashOutputs`、旧TX output脚本`scriptCode`，然后再从`scriptCode`中取出最后一个字节，这就是旧TX中计数器的值，存在变量`counter`里。



### 计算符合合约规则的新output hash值

```c++
bytes scriptCode_ = scriptCode[: scriptLen - 1] ++ num2bin(counter + 1, 1);
Sha256 hashOutputs_ = hash256(num2bin(amount, 8) ++ Util.writeVarint(scriptCode_));
```

前面提到，新TX output脚本的逻辑部分与旧TX一样，只是数据部分的值增加了1。所以，把旧TX output脚本`scriptCode`的逻辑部分保留，把数据部分`counter`加1，并放到逻辑部分后面，这就是符合合约规则的新TX output脚本了。第一行代码就是这个作用，新脚本存在了变量`scriptCode_`中。

再把`scriptCode_`与新TX output的satoshis数量`amount`合并在一起做sha256 hash，就得到了新TX output的hash值`hashOutputs_`。这里仍然暂时忽略对`amount`参数的解释。



### 检查计算出的hash值与实际的新TX output的hash值是否一致

```c++
require(hashOutputs == hashOutputs_);
```

前面已经从`sighashPreimage`参数中解析出了实际的新TX output hash值`hashOutputs`，也计算出了符合合约规则的预期hash值`hashOutputs_`。只要比较这两个值是否相等，就可以确定实际的TX output是否符合规则。如果不符合规则，那么旧TX的UTXO就无法被花费。

这样，整个合约就完成了。这个合约已经部署在了BSV的测试网络上：[0](https://test.whatsonchain.com/tx/5bde01982a262beb5f438ca36ee27ca75467ac890183b329e2fd5fcb16b488cf), [1](https://test.whatsonchain.com/tx/1f3c9b9dfacc4ff485d6fecf01c7dd1e5d8d8493d24b259a1d57b4e759eaf926), [2](https://test.whatsonchain.com/tx/34418a69f2dee4b2e7263f4d56a8fd5bd5e301a6e2120fde50672eed7537cb0e) ...



## 看明白了吗？

没看明白再看一遍吧。

下一篇我们将填上这篇中挖的坑：如何保证`sighashPreimage`参数的真实性。



----

参考资料：

* [Stateful Smart Contracts on Bitcoin SV](https://medium.com/coinmonks/stateful-smart-contracts-on-bitcoin-sv-c24f83a0f783)
* [OP_PUSH_TX](https://medium.com/@xiaohuiliu/op-push-tx-3d3d279174c1)

----

上一篇：[看似不可能完成的任务](https://github.com/corper/bsv-smart-contract/blob/master/01-mission-impossible-seemingly.md)

下一篇：[神奇的OP_CHECKSIG](https://github.com/corper/bsv-smart-contract/blob/master/03-magical-op-checksig.md)