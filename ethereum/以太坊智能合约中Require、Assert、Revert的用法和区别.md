# 以太坊智能合约中Require(), Assert(), Revert()的用法和区别

内容总结自：[https://medium.com/blockchannel/the-use-of-revert-assert-and-require-in-solidity-and-the-new-revert-opcode-in-the-evm-1a3a7990e06e ](https://medium.com/blockchannel/the-use-of-revert-assert-and-require-in-solidity-and-the-new-revert-opcode-in-the-evm-1a3a7990e06e)。若有不准确的地方还请指正！

## 前言

在Solidity0.4.10之前，if...throw普遍利用于判断一个条件是否满足，如果不满足则终断运行。但这throw了之后它会撤回所有的状态转变，用光你所有的gas，所以这并不是一个好的操作。

之后，assert(), require(), and revert() 三个函数代替了if...throw的功能，并对gas有了更好的处理。原文章中提到的例子：

if(msg.sender != owner) { revert(); }

assert(msg.sender == owner);

require(msg.sender == owner);

这三个函数在功能上是与if(msg.sender != owner) { throw; }是等价的。下面具体说说这三个函数的区别。



### Require and assert

同样作为判断一个条件是否满足的函数，require会退回剩下的gas，而assert会烧掉所有的gas。对于两个函数应该在什么情况下使用，这里引用一段原文：

```

The require function should be used to ensure valid conditions, such as inputs, or contract state variables are met, or to validate return values from calls to external contracts. If used properly, analysis tools can evaluate your contract to identify the conditions and function calls which will reach a failing assert. Properly functioning code should never reach a failing assert statement; if this happens there is a bug in your contract which you should fix.

```

### Revert

revert的用法和throw很像，也会撤回所有的状态转变。但是它有两点不同：

1. 它允许你返回一个值；

2. 它会把所有剩下的gas退回给caller

调用起来就像这样子：

revert(‘Something bad happened’);

require(condition, ‘Something bad happened’);


### 适合用Require的时候：

验证一个用户的输入是否合法：ie.require(input < 20);

验证一个外部协议的回应：require(external.send(amount));

判断执行一段语句的前置条件： ie.require(block.number > SOME_BLOCK_NUMBER) or require(balance[msg.sender]>=amount)；

require应该被最常使用到；

一般用于函数的开头处。

### 适合用Revert的时候：

和require（）应用场景差不多，适合用在逻辑复杂的情况下。

### 适合用Assert的时候：

检查有没有上溢或者是下溢： ie. c = a+b; assert(c > b)

检查常数： ie. assert(this.balance >= totalSupply);

在完成变化后检查状态

避免本不应该发生的情况出现，如程序的bug

assert不应该被经常利用到；

一般用于函数结尾处





## 结论

这些函数是安全性检查工具库中很强大的工具。知道如何以及何时使用这些函数不仅能帮助你的代码免受攻击，而且会使代码更加对用户友好，更加面向未来变化。

## 参考

- [https://www.jianshu.com/p/4560cf887f71](https://www.jianshu.com/p/4560cf887f71)
- [https://ethfans.org/posts/when-to-use-revert-assert-and-require-in-solidity](https://ethfans.org/posts/when-to-use-revert-assert-and-require-in-solidity)
- [https://www.cnblogs.com/tinyxiong/p/8735466.html](https://www.cnblogs.com/tinyxiong/p/8735466.html)