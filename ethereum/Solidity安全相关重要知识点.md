# Solidity安全相关重要知识点


# 以太币转账三种方式对比

call.value()()  VS  send()  VS  transfer()

|    Method      |   gas消耗    | returns on fail |SafeLevel |
|    :------:    |   :------:     |    :------:    |   :------:   |
| call.value()() | 所有gas       |  FALSE         |   Low   |
| send()         | 限制到2300    |  FALSE         |   Medium | 
| transfer()     | 限制到2300    |  Throw         |   High   |


如上图所示，其中最安全的方式当属 transfer()，一旦转账失败，transfer() 会抛出异常直接触发 revert() 事件，而另外两者不会，需要开发人员手动处理返回值。send() 与 transfer() 唯一的区别也在于返回值，通常我们可以认为addr.transfer(v) 就相当于require(addr.send(v))。


而call.value() 与另外两者一个最明显的区别在于gas的限制上面，call.value()允许消耗掉所有的gas。但另外两种方式由于gas消耗限制到2300，不足以完成递归调用，这也是能够避免 reentrancy 攻击的原因.

所以，建议开发人员，在ERC20合约开发过程中，如果遇到以太币转出的情况，如非必要，尽量选择使用transfer()函数来完成。

# 底层调用三种方式对比


3个底层调用 call(), delegatecall(), callcode()：

call 的外部调用上下文是外部合约  
delegatecall、callcode 的外部调用上下是调用合约上下文  
简单的用图表示就是：  
![](https://p2.ssl.qhimg.com/t01b536ee36490b90dd.png)  
合约 A 以 call 方式调用外部合约 B 的 func() 函数，在外部合约 B 上下文执行完 func() 后继续返回 A 合约上下文继续执行；而当 A 以 delegatecall 方式调用时，相当于将外部合约 B 的 func() 代码复制过来（其函数中涉及的变量或函数都需要存在）在 A 上下文空间中执行。


– call()

call() 用于 Solidity 进行外部调用，例如调用外部合约函数< address >.call(bytes4(keccak("somefunc(params)"), params))，外部调用 call() 返回一个 bool 值来表明外部调用成功与否：

```
pragma solidity ^0.4.24; 

contract MyCall{

	event CallResult(bool);
    
    function extCall(address_sc){
    
    	bool result=_sc.call(bytes4(keccak236("funcl()")));
        CallResult(result);
    }
}

contract SC{

	// 	通过call() 进行外部调用
    //  外部调用不管是revert()或者throw；还是发生其他异常，都只会返回false值给调用方，表明外部调用失败或者发生了错误
	function funcl(){
    	revert();
    }
}

```

– delegatecall()

除了 delegatecall() 会将外部代码作直接作用于合约上下文以外，其他与 call() 一致，同样也是只能获取一个 bool 值来表示调用成功或者失败（发生异常）。

– callcode()

callcode() 其实是 delegatecall() 之前的一个版本，两者都是将外部代码加载到当前上下文中进行执行，但是在 msg.sender 和 msg.value 的指向上却有差异。

例如 Alice 通过 callcode() 调用了 Bob 合约里同时 delegatecall() 了 Wendy 合约中的函数，这么说可能有点抽象，看下面的代码：

```
pragma solidity ^0.4.24;

contract Bob{
    
    uint256 public n;
    address public sender;
    
    function callcodeWendy(address _windy, uint256 _n){
        // sender will be Bob
        _windy.callcode(bytes4(keccak256("setN(uint256)")),_n);
    }
    
        function delegatecallWendy(address _windy, uint256 _n){
        // sender will be who Bob.delegatecallWendy-> Alice
        _windy.delegatecall(bytes4(keccak256("setN(uint256)")),_n);
    }
}

contract Wendy{
    
    uint256 public n;
    address public sender;
    
    function setN (uint256 _n){
        n=_n;
        sender=msg.sender;
    }
}

```

如果还是不明白 callcode() 与 delegatecall() 的区别，可以将上述代码在 remix-ide 里测试一下，观察两种调用方式在 msg.sender 和 msg.value 上的差异。

# 错误处理

assert(bool condition): 如果条件不满足就抛出—用于内部错误。

require(bool condition): 如果条件不满足就抛掉—用于输入或者外部组件引起的错误。

revert(): 终止运行并恢复状态变动。

# 数据位置(Data location)

首先，为了更好地理解EVM(以太坊虚拟机)中的内存管理机制，我们还是先来看看Solidity中的基本数据类型。

- 值类型：

包括整型(int/uint)，地址型(address)，布尔(Booleans)，定长字节数组(fixed byte arrays)，枚举类型(Enums)...在内的基本数据类型，在合约中都是以值的类型进行传递(赋值)；

- 引用类型：

而那些复杂类型，占用空间较大的。考虑到在拷贝到内存时成本比较大。所以考虑通过引用传递。比如，不定长字节数组（bytes），字符串（string），数组（Array），结构体（Struts）。

## 状态变量的存储模型(Layout of State Variables in Storage)

* 大小固定的变量（除了映射，变长数组以外的所有类型）在存储(storage)中是依次连续从位置0开始排列的。如果多个变量占用的大小少于32字节，会尽可能的打包到单个storage槽位里，具体规则如下：
  - 在storage槽中第一项是按低位对齐存储（lower-order aligned）（译者注：意味着是大端序了，因为是按书写顺序。）。 
  - 基本类型存储时仅占用其实际需要的字节。
  - 如果基本类型不能放入某个槽位余下的空间，它将被放入下一个槽位。
  - 结构体和数组总是使用一个全新的槽位，并占用整个槽(但在结构体内或数组内的每个项仍遵从上述规则)

优化建议

- 当使用的元素占用少于32字节，你的合约的gas使用也许更高。这是因为EVM每次操作32字节。因此，如果元素比这小，EVM需要更多操作来从32字节减少到需要的大小。

- 因为编译器会将多个元素打包到一个storage槽位，这样就可以将多次读或写组合进一次操作中，只有在这时，通过缩减变量大小来优化存储结构才有意义。当操作函数参数和memory的变量时，因为编译器不会这样优化，所以没有上述的意义。

- 最后，为了方便EVM进行优化，尝试有意识排序storage的变量和结构体的成员，从而让他们能打包得更紧密。比如，按这样的顺序定义，uint128, uint128, uint256，而不是uint128, uint256, uint128。因为后一种会占用三个槽位。

* 非固定大小
  - 结构体和数组里的元素按它们给定的顺序存储。
  - 由于它们不可预知的大小。映射和变长数组类型，使用Keccak-256哈希运算来找真正数据存储的起始位置。这些起始位置往往是完整的堆栈槽。
  - 映射和动态数组根据上述规则在位置p占用一个未满的槽位(对映射里嵌套映射，数组中嵌套数组的情况则递归应用上述规则)。对一个动态数组，位置p这个槽位存储数组的元素个数(字节数组和字符串例外，见下文)。而对于映射，这个槽位没有填充任何数据(但这是必要的，因为两个挨着的映射将会得到不同的哈希值)。数组的原始数据位置是keccak256(p)；而映射类型的某个键k，它的数据存储位置则是keccak256(k . p)，其中的.表示连接符号。如果定位到的值以是一个非基本类型，则继续运用上述规则，是基于keccak256(k . p)的新的偏移offset。
  - bytes和string占用的字节大小如果足够小，会把其自身长度和原始数据存在当前的槽位。具体来说，如果数据最多31位长，高位存数据(左对齐)，低位存储长度lenght 2。如果再长一点，主槽位就只存lenght 2 + 1。原始数据按普通规则存储在keccak256(slot)

数据的存储位置的计算规则就是：

keccak256(bytes32(key) + bytes32(position))

此处key即为映射的key，而position也就是该变量本来的位置，根据此式我们可以手动算出变量存储位置。
对于uint、address、bytes、array、struct、mapping等类型的数据结构
* 目前通过实验只有mapping有key的概念，因此在算mapping的数据结构存储地址的时候，可以根据keccak256(bytes32(key) + bytes32(position))计算。
* 对于uint、address、bytes、array、struct等没有key概念等数据结构，直接通过keccak256(bytes32(position))计算。

强制的数据位置(Forced data location)

* 外部函数(External function)的参数(不包括返回参数)强制为：calldata
* 状态变量(State variables)强制为: storage

默认数据位置（Default data location）

* 函数参数及返回参数：memory
* 复杂类型的局部变量：storage

因此禁止如下书写代码：
```
contract test {
    uint256 public a=0x123;
    uint256 public b=0x456;
    uint256 public c=0x789;

    function testforfun(){
    uint256[3] z;
    z[0]=1;
    z[1]=2;
    z[2]=3;
    }
}
```
这样写代码，将会导致全局变量被覆盖。

**重点**：复杂类型包括数组和映射，在函数中定义为局部变量的时候也都需要添加memory，不然将会导致某些全局变量会被覆盖。
正确的实例代码如下：

```
contract test {
    uint256 public a=0x123;
    uint256 public b=0x456;
    uint256 public c=0x789;

    function testforfun(){
    uint256[3] memory z;
    z[0]=1;
    z[1]=2;
    z[2]=3;
    }
}
```

参考：

- [http://www.freebuf.com/articles/blockchain-articles/175237.html](http://www.freebuf.com/articles/blockchain-articles/175237.html)

# 参考
- [https://www.sohu.com/a/232608666_100064654](https://www.sohu.com/a/232608666_100064654)
- [https://www.cnblogs.com/tinyxiong/p/8084477.html](https://www.cnblogs.com/tinyxiong/p/8084477.html)