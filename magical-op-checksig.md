# BSV智能合约（三）：神奇的OP_CHECKSIG

[上一篇](https://github.com/corper/bsv-smart-contract/blob/master/02-scrypt-code-analysis.md)我们分析了计数器合约的代码，其中最关键的问题是：**如何保证`sighashPreimage`参数的真实性**。要弄清楚这个问题，先要从比特币脚本的操作码`OP_CHECKSIG`说起。

[OP_CHECKSIG](https://wiki.bitcoinsv.io/index.php/OP_CHECKSIG)用于验证ECDSA签名。ECDSA签名的验证需要如下三个参数：

1. 公钥：签名私钥对应的公钥。
2. 数据：被签名的数据的Hash值。
3. 签名。

如果签名是由公钥对应的私钥针对数据进行签署的，那么签名验证成功，否则失败。而OP_CHECKSIG验证的ECDSA签名，只需要公钥和签名两个参数。为什么没有“被签名数据”参数呢？因为在OP_CHECKSIG中，“被签名数据”参数是一组固定数据的Hash值。这组固定数据是这样构成的：

> 1. nVersion of the transaction (4-byte little endian)
> 2. hashPrevouts (32-byte hash)
> 3. hashSequence (32-byte hash)
> 4. outpoint (32-byte hash + 4-byte little endian)
> 5. **scriptCode of the input (serialized as scripts inside CTxOuts)**
> 6. value of the output spent by this input (8-byte little endian)
> 7. nSequence of the input (4-byte little endian)
> 8. **hashOutputs (32-byte hash)**
> 9. nLocktime of the transaction (4-byte little endian)
> 10. sighash type of the signature (4-byte little endian)

上述5、8两项是不是看着有点眼熟？是的，这就是上一篇里从sighashPreimage参数里解析出来的两个数据。其实，上面这组数据就是sighashPreimage。只是，这里的sighashPreimage是验证签名时，比特币的实现代码（如全节点代码）从老TX（被花费的TX）和新TX中获取的，是真实的，我们称之为`真实sighashPreimage`。而计数器合约中的sighashPreimage是传入合约的解锁参数之一，不一定与真实的一致，我们称之为`传入sighashPreimage`。而如何保证`传入sighashPreimage`与`真实sighashPreimage`是一样的，正是计数器合约要解决的关键问题。



接下是最巧妙的部分，计数器合约用比特币脚本做了如下两步操作：

1. 选择一个对公私钥privateKey和publicKey，用私钥对`传入sighashPreimage`进行签名，得到签名数据，用公式表达为： `signature = sig(privateKey, hash(传入sighhashPreimage))`
2. 用上述公钥对上述计算出的签名进行OP_CHECKSIG签名校验，用公式表达为：`checkResult = op_checksig(publicKey, hash(真实sighashPreimage), signature)`。（再次强调，OP_CHECKSIG操作码规定，被签名的数据一定是`hash(真实sighashPreimage)`，否则OP_CHECKSIG的行为就不符合协议，相当于实现OP_CHECKSIG的代码错了。）

如果第2步的签名通过，也就是说`checkResult`为`true`，那么`传入sighashPreimage`与`真实sighashPreimage`相同。计数器合约正是这样巧妙地运用OP_CHECKSIG操作码来验证了`传入sighashPreimage`的真实性。

从上面的计算过程我们可以知道，这里公私钥对的选择并不重要，部署合约时随机生成一对就可以。因为公私钥对要写到合约脚本里公开，所以千万不要选择可以花费币的私钥。



这种检测`传入sighashPreimage`真实性的方案有个专有名字，叫OP_PUSH_TX。虽然以OP开头，但这并不是一个BSV协议中的基本操作码。可以把它理解为用基本操作码编写的一个高层函数。sCrypt语言中用如下两行代码来实现OP_PUSH_TX：

```c++
Tx tx = new Tx();
require(tx.validate(sighashPreimage));
```

该方案及相关技术已由nChain公司申请了专利。



用脚本直接计算ECDSA签名，用OP_CHECKSIG验证传入数据的真实性。这是十年来在比特币上开的最大脑洞之一，脑洞的另一端也许是一个新的宇宙。

----

参考资料

1. [Bitcoin SV wiki OP_CHECKSIG](https://wiki.bitcoinsv.io/index.php/OP_CHECKSIG)
2. [BSV & BCH signature verification](https://github.com/bitcoin-sv/bitcoin-sv/blob/master/doc/abc/replay-protected-sighash.md#digest-algorithm)
3. [OP_PUSH_TX](https://medium.com/@xiaohuiliu/op-push-tx-3d3d279174c1)