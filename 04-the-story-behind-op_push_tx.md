# OP_PUSH_TX背后的故事

* 该技术由nChain的前研究员[Ying Chan](https://www.linkedin.com/in/ying-chan-10077651/?originalSubdomain=uk)在2016/2017年提出。
* nChain有大约至少9项专利与该技术有关。
* 2018年BCH与BSV分叉前，nChain曾提出过该技术，但遭到质疑，方案被认为可能行不通，并且会导致脚本体积过大。2020年初sCrypt团队实现了该方案，证明这个方案是可行的，并且在BSV上TX体积（约2K字节）并不是问题。
* BCH也有类似的技术方案，不过并不是通过原有操作码OP_CHECKSIG来实现的，而是[通过引入新操作码OP_CHECKDATASIG来实现](https://cashscript.org/docs/language/globals#global-covenant-variables)。是否引入新操作码OP_CHECKDATASIG是2018年底造成BCH和BSV直接分裂的原因之一。

----

参考资料：

* [The story behind the OP_PUSH_TX technique](https://coingeek.com/the-story-behind-the-op_push_tx-technique/)
* [A Case For Malfix](https://honest.cash/v2/cpacia/a-case-for-malfix-4436)

----

上一篇：[神奇的OP_CHECKSIG](https://github.com/corper/bsv-smart-contract/blob/master/03-magical-op-checksig.md)

