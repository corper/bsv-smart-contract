# BSV智能合约（一）：看似不可能完成的任务

**如果你转出了一笔BSV，你还能控制这些BSV如何被使用吗？**

可能很多人的答案是：不能。一旦币被转出，就意味着失去了控制，收到币的人，想怎么用就怎么用。而实际上，跟很多人的想法相反，BSV是可以通过脚本来控制转出去的币的，这些币只能按照脚本已经规定好的方式使用，而不能用做其他用途。

让我们看一个具体的例子，一个“计数器合约”。这个合约非常简单，就是把自己的被调用次数记录在区块链上。具体需求如下：

1. 有一个UTXO被称作“计数器合约”，该UTXO记录了计数器的初始值：0。
2. 当“计数器合约”里的币被花费时，必须生成一个新的“计数器合约”UTXO，并将计数器值加1，记录在这个新UTXO中。
3. 规则2的执行由矿工保证。也就是说，如果没有符合规则的新“计数器合约”UTXO生成，就无法花费合约中的币。

![counter contract tx flow](https://bico.media/b04a2c2774cfab56782b689bc8710f5fefe8d7287a5cf338dde22a47c658b637)



如果你理解了上述“计数器合约”的规则，也许你会觉得这样的合约无法在比特币上实现，尤其是规则2，简直匪夷所思：难道一个UTXO在被花费时，用自身脚本就可以控制新UTXO的生成吗？

是的，可以。2020年2月4日[BSV的Genesis版本](https://github.com/bitcoin-sv/bitcoin-sv/releases/tag/v1.0.0)升级后，恢复了更多比特币早期的操作码，放开了更多脚本限制，使得BSV的脚本更加完善，可以完成这个看似不可完成的任务。

在下一篇，我们将介绍如何实现这个合约。在下一篇中登场的是主角将是BSV上的高级语言[sCrypt](https://scryptdoc.readthedocs.io/en/latest/)。

----

参考资料：

* [Stateful Smart Contracts on Bitcoin SV](https://medium.com/coinmonks/stateful-smart-contracts-on-bitcoin-sv-c24f83a0f783)
