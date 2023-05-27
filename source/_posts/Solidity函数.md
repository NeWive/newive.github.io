---
title: Solidity函数
date: 2023-05-16 18:31:17
tags: 
- Solidity
- Smart Contract
---

## Get 函数

编译器自动为public类型的状态变量创建Get函数。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.16 <0.9.0;

contract C {
    uint public data = 42;
}

contract Caller {
    C c = new C();
    function f() public view returns (uint) {
        data = 3; // internal access
        return this.data(); // external access
    }
}
```

自动生成的Get函数具有external属性。

如果是一个public的数组则只能通过get函数获取单个元素，其目的是防止返回整个数组导致消耗过多gas。

## 装饰器

装饰器例子

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.0 <0.9.0;

contract Owned {
    address payable owner;
    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner {
        require(msg.sender == owner, "Only owner can call this function");
        _;
    }
}

contract Destructible is Owned {
    function destroy() public onlyOwner {
        selfdestruct(owner);
    }
}

contract Priced {
    // 接收参数
    modifier costs(uint price) {
        if (msg.value >= price) {
            _;
        }
    }
}

contract Register is priced, destructible {
    mapping(address => bool) registeredAddresses;
    uint price;

    constructor(uint initialPrice) { price = initialPrice; }

    function register() public payable costs(price) {
        registeredAddresses[msg.sender] = true;
    }

    function changePrice(uint price_) public onlyOwner {
        price = price_;
    }
}

contract Mutex {
    bool locked;
    modifier noReentrancy() {
        require(
            !locked,
            "Reentrant call."
        );
        locked = true;
        _;
        locked = false;
    }

    function f() public noReentrancy returns (uint) {
        (bool success,) = msg.sender.call("");
        require(success);
        return 7;
    }
}
```

## 对`view`与`pure`类型函数的一点说明

下列情况被认为是修改状态:

- 写入状态变量
- 触发事件
- 创建其他合约
- 使用`selfdestruct`
- 通过调用发送以太币
- 调用没有被标记为`view`或`pure`的函数
- 使用底层调用
- 使用包含特定操作码的内联汇编

下列情况被认为是读取状态

- 读取状态变量
- 访问`address(this).balance`或`<address>.balance`
- 访问`tx`、`block`等全局变量的成员(除`msg.sig`与`msg.data`外)
- 调用任何没有被标记为`pure`的函数
- 使用包含特定操作码的内联汇编

## selector

调用函数时，具体调用的函数信息会被拼装成calldata，calldata的前四个字节就是这个甘函数的selector，selector是这个函数的唯一标识。

通过拼接selector与函数形参可以在合约中得到calldata，并在合约中通过call的方式调用另外一个合约中的方法，从而实现合约间的调用。

### 获取selector的方式

#### 使用`abi.encodeWithSignature` 

通过拼接selector与函数形参可以在合约中得到calldata，并在合约中通过call的方式调用另外一个合约中的方法，从而实现合约间的调用。

在一个合约中一个函数的4字节selector可以通过`abi.encodeWithSignature`获取。

**注意，其获取参数的方式时通过传入的字符串计算哈希，所以不可以有多余的空格等符号，形参也必须与原函数保持一致，uint256与uint也会导致不同的结果。**

```solidity
bytes memory transferSelector = abi.encodeWithSignature("transfer(address, uint256)");

// 调用
addr.call(transferSelector, 0xSomeAddress, 100);

// 更函数式的写法
addr.call(abi.encodeWithSignature("transfer(address, uint256)"), 0xSomeAddress, 100);
```

#### keccak256

```solidity
// 获取哈希时仅处理函数名和形参而不必考虑返回值
bytes4(keccak256(bytes("transfer(address,uint256)")));
```

#### 直接获取

使用`this.transfer.selector`

#### 使用`abi.encodeCall`方式

```solidity
interface IERC20 {
	function transfer(address, uint) external;
}

function encodeCall(address to, uint amount) external pure returns (bytes memory) {
	return abi.encodeCall(IERC20.transfer, (to, amount));
}
```

## 调用外部合约中的函数

合约间调用的方式主要包括：

- 通过合约实例
- 使用call调用合约
- 使用delegate调用合约

### call

call时一种底层调用合约的方式，可以在合约内调用其他合约

```solidity
(bool success, bytes memory data) = addr.call{value: ValueAmt, gas: gasAmt}(abi.encodeWithSignature("foo(string,uint256)"), ...args);
```

返回值：

- `success`: 执行结果，必须要校验是否成功，失败务必要回滚
- `data`: 调用函数的返回值，是打包的字节序，需要重新解析才能得到原始返回值。

**当调用fallback方式给合约转ether时建议使用call而不是transfer或send**

以下是通过fallback的方式转账

```solidity
(bool success, bytes memory data) = addr.call{value: 10}("");
```

注意，当调用的方法不存在，且合约又未实现fallback时，**交易会调用成功，但是第一个参数为：false**，所以使用call调用后一定要检查success状态。

```solidity
(bool success, bytes memory data) = _addr.call(abi.encodeWithSignature("doesNotExist()"));
```

### delegatecall

当A合约通过delegatecall调用的B合约的函数时，使用的是A的上下文，包括A的状态变量，msg.value, msg.sender等。

因此使用delegatecall的前提是A、B具有相同的状态变量。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

contract Callee {
    uint public x;
    uint public value;

    function setX(uint _x) public returns (uint) {
        x = _x;
        return x;
    }

    function setXandSendEther(uint _x) public payable returns (uint, uint) {
        x = _x;
        value = msg.value;

        return (x, value);
    }
}

contract Caller {
    // 直接在参数中进行实例化合约
    function setX(Callee _callee, uint _x) public {
        uint x = _callee.setX(_x);
    }

    // 传递地址，在内部实例化callee合约
    function setXFromAddress(address _addr, uint _x) public {
        Callee callee = Callee(_addr);
        callee.setX(_x);
    }

    // 调用方法，并转ether
    function setXandSendEther(Callee _callee, uint _x) public payable {
        (uint x, uint value) = _callee.setXandSendEther{value: msg.value}(_x);
    }
}
```

## fallback机制

fallback可以理解为一个可选实现的函数接口，无参数，无返回值。

会被调用的情况：

- 当被调用的方法不存在时fallback会被调用
- 当向合同转ether而合同不存在receive函数时
- 当向合同转ether但是msg.data不为空时，即使receive存在

当使用transfer或者send转账时fallback的gaslimit限定为2300gas

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

contract Fallback {
	event Log(uint gas);
	
	fallback() external payable {
		emit Log(gasLeft());
	}
	
	function getBalance() public view returns (uint) {
		return address(this).balance;
	}
}

contract SendTOFallback {
	function transferToFallback(address payable _to) public payable {
		_to.transfer(msg.value);
	}
	
	function callFallback(address payable _to) public payable {
		        // Log event:  "gas": "6110"
        (bool sent, ) = _to.call{value: msg.value}("");
        require(sent, "Failed to send Ether");
	}
	
	function callNoExistFunc(address payable _to) public payable {
        // call no exist funtion will call fallback by default 
        // Log event:  "gas": "5146"
        (bool sent, ) = _to.call{value: msg.value}(abi.encodeWithSignature("noExistFunc()"));
        require(sent, "Failed to call");
    }
}
```

## Library

库与合约类似，限制：不能在库中定义状态变量，不能向库地址中转入ether；

库有两种存在形式：

1. 内嵌（embedded）：当库中所有的方法都是internal时，此时会将库代码内嵌在调用合约中，不会单独部署库合约；
2. 链接（linked）：当库中含有external或public方法时，此时会单独将库合约部署，并在调用合约部署时链接link到库合约。
   1. 可以复用的代码可以编写到库中，不同的调用	者可以linked到相同的库，因此会更加节约gas；
   2. 对于linked库合约，调用合约使用delegatecall进行调用，所以上下文为调用合约；
   3. 部署工具（如remix）会帮我们自动部署&链接合约库。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

// 1. 只有internal方法，会内嵌到调用合约中
library SafeMath {
    function add(uint x, uint y) internal pure returns (uint) {
        uint z = x + y;
        require(z >= x, "uint overflow");

        return z;
    }
}

library Math {
    function sqrt(uint y) internal pure returns (uint z) {
        if (y > 3) {
            z = y;
            uint x = y / 2 + 1;
            while (x < z) {
                z = x;
                x = (y / x + x) / 2;
            }
        } else if (y != 0) {
            z = 1;
        }
        // else z = 0 (default value)
    }
}

contract TestSafeMath {
      // 对uint类型增加SafeMath的方法，
      // 1. 后续定义的uint变量就会自动绑定SafeMath提供的方法: uint x;
      // 2. 这个变量会作为第一个参数传递给函数: x.add(y);
// 注意
    using SafeMath for uint;
// 注意
    uint public MAX_UINT = 2**256 - 1;

      // 用法1：x.方法(y)
    function testAdd(uint x, uint y) public pure returns (uint) {
        return x.add(y);
    }

      // 用法2：库.方法(x)
    function testSquareRoot(uint x) public pure returns (uint) {
        return Math.sqrt(x);
    }
}

// 2. 存在public方法时，会单独部署库合约，并且第一个参数是状态变量类型
library Array {
      // 修改调用者状态变量的方式，第一个参数是状态变量本身
    function remove(uint[] storage arr, uint index) public {
        // Move the last element into the place to delete
        require(arr.length > 0, "Can't remove from empty array");
        arr[index] = arr[arr.length - 1];
        arr.pop();
    }
}

contract TestArray {
// 注意
    using Array for uint[];
// 注意
    uint[] public arr;

    function testArrayRemove() public {
        for (uint i = 0; i < 3; i++) {
            arr.push(i);
        }

        arr.remove(1);

        assert(arr.length == 2);
        assert(arr[0] == 0);
        assert(arr[1] == 2);
    }
}
```

## 节约gas的一些技巧

- 使用calldata替换memory
- 将状态变量加载到memory中
- 使用++i替换i++
- 对变量进行缓存