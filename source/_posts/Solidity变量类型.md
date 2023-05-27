---
title: Solidity变量类型与全局变量
date: 2023-05-16 14:51:06
tags:
- Solidity
- Smart Contract
---
Solidity是静态变量语言。`undefined`或`null`的概念不存在于Solidity中。
同时所有初始未进行赋值的变量都有默认值0。
处理异常时最好使用`revert`来回滚整个交易或返回一个`bool`类型进行说明。

## 变量的可见性和可修改性

### 可见性(具体解释详见函数的可见性)

- `public`
- `private`
- `internal`

### 可修改性

- `constant`: 必须有默认值
- `immutable`: 不必有默认值但只能在构造函数里初始化

## 基本类型

- `bool`

- `int` / `uint`: 类型从`uint8`以8位为一步增长到`uint256`，不加位数的`int`或`uint`默认为256位
  - 使用`type(X).max`和`type(X).min`获取极值，其中X为变量类型。
  - 默认情况下如果数学计算的结果发生了溢出则会通过失败断言的方式进行回滚。

- `address`: 保存一个长度为20字节的值，等同于以太坊地址的长度
- `address payable`: 相比于`address`增加了`transfer`与`send`两个成员函数，代表该地址是一个可以发送以太币的地址。
  - `address payable`可以隐式转换为`address`，反之则不行，需要使用`payable(<address>)`进行转换。

### `address`类型的成员函数与变量

- `.balance`: `uint256`, 单位为Wei
- `.transfer(uint256 amount)`: `address payable`的成员函数，交易失败会回滚
- `.send(uint256 amount) returns (bool)`: 同上，但失败会返回false，不会回滚
- `.call(bytes memory) returns (bool, bytes memory)`: 底层调用，一般用于调用外部函数，参数为函数的signature，*使用时必须要检查是否成功*
- `.delegatecall(bytes memory) returns (bool, bytes memory)`: 同上
- `.staticcall(bytes memory) returns (bool, bytes memory)`: 同上

使用call时会绕过类型检查、函数存在性检查和参数打包，所以应该尽可能避免使用。
send函数具有潜在的风险，如果交易的调用栈深度超过1024时或gas消耗完毕时会失败。所以尽可能使用transfer或检查send的执行结果。
如果调用的帐户不存在，作为 EVM 设计的一部分，低级函数 call、delegatecall 和 staticcall 将返回 true 作为它们的第一个返回值。如果需要，必须在调用之前检查帐户是否存在。

## 合约类型

合约类型可以隐式转换成它们继承的类型，可以被显式转换成地址类型，也可以从地址类型显式转换成合约类型。
其中与`address payable`的转换只能通过`payable(address(x))`的方式实现，x为某一个合约的实例。

### 通过new创建合约

```solidity
pragma solidity >=0.7.0 <0.9.0;
contract D {
    uint public x;
    constructor(uint a) payable {
        x = a;
    }
}

contract C {
    D d = new D(4); // will be executed as part of C's constructor

    function createD(uint arg) public {
        D newD = new D(arg);
        newD.x();
    }

    function createAndEndowD(uint arg, uint amount) public payable {
        // Send ether along with the creation
        D newD = new D{value: amount}(arg);
        newD.x();
    }
}
```

## 枚举类型

```solidity
//SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.0;

contract Enum {
    enum ActionChoices {l, r, s};
    ActionChoices choice;
    ActionChoices constant defaultChoice = ActionChoices.l;

    function setGoStraight() public {
        choice = ActionChoices.GoStraight;
    }

    function getChoice() public view returns (ActionChoices) {
        return choice;
    }

    // 与uint转化需显式
    function getDefaultChoice() public pure returns (uint) {
        return uint(defaultChoice);
    }

    // 获取极值
    function getLargestValue() public pure returns (ActionChoices) {
        return type(ActionChoices).max;
    }

    function getSmallestValue() public pure returns (ActionChoices) {
        return type(ActionChoices).min;
    }
}
```

## 函数类型

### 函数类型的修饰符

#### 可见性

- `public`: 表示在任何情况下都可以访问
- `private`: 只有当前类可以访问，子类无法访问，外部无法访问
- `internal`: 当前类及其子类可以访问，外部无法访问
- `external`: 只有外部可以访问

默认情况下函数类型是`internal`，可以省略。

`public`与`external`函数具有以下成员变量
- `address`: 函数的地址
- `selector`: 返回ABI function selector，表示一个函数的特征。

#### 状态的修改性

主要用于约束开发的规范性（个人感觉）。

- `pure`: 不涉及状态变量的函数
- `view`: 函数的逻辑中只有对状态变量的读取，没有对状态变量的修改
- `payable`: 涉及以太币交易的函数

函数A到B之间的隐式转换只有在参数、返回值等完全相同且A的状态可变性比B的更严格的条件下才被允许。

- `pure`可以被转化为`view`和`non-payable`
- `view`可以被转化为`non-payable`
- `payable`可以被转化为`non-payable`

另外`non-payable`的函数会拒绝发送给它的以太币，同时拒绝以太币比不拒绝以太币更具有限制性所以可以用一个`payable`函数覆盖`non-payable`函数。

带有`calldata`类型参数的`external`函数不能被带有`calldata`类型参数的函数类型所兼容。例如`function (string calldata) external`类型不能指向任何函数，而`function (string memory) external`可以同时指向`function f(string memory) external {}`和`function g(string calldata) external {}`。这是因为对于这两个位置，参数以相同的方式传递给函数。调用者不能直接将它的calldata传递给一个外部函数。

### 函数调用

`this`在构造函数中不可以被使用。
当调用函数时可以写明Wei的数量和调用发送的gas，`{value: 10, gas: 1000}`。*但是不建议写明gas值*。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.2 <0.9.0;

contract InfoFeed {
    function info() public payable returns (uint ret) { return 42; }
}

contract Consumer {
    InfoFeed feed;
    function setFeed(InfoFeed addr) public { feed = addr; }
    function callFeed() public { feed.info{value: 10, gas: 800}(); }
}
```

同时函数调用的参数可以根据变量名指明。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.0 <0.9.0;

contract C {
    mapping(uint => uint) data;

    function f() public {
        set({value: 2, key: 3});
    }

    function set(uint key, uint value) public {
        data[key] = value;
    }
}
```

### 解构赋值

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.5.0 <0.9.0;

contract C {
    uint index;

    function f() public pure returns (uint, bool, uint) {
        return (7, true, 2);
    }

    function g() public {
        // 用类型声明并从返回的元组分配的变量
        // 不必指定所有元素（但数量必须匹配）
        (uint x, , uint y) = f();
        // 交换值的常用技巧——不适用于非值存储类型。
        (x, y) = (y, x);
        // 可以省略
        (index, , ) = f(); // Sets the index to 7
    }
}
```

## 引用类型

引用类型包括数组、结构体和映射。当使用引用类型时需要显式写明数据位置。

### 数据位置

- `memory`: 作用域在一个函数调用内部
- `storage`: 永久存储在区块链中，开销较大
- `calldata`: 包含函数参数的特殊位置，且是只读的

更改数据位置的赋值或类型转换将始终引发深拷贝，而同一数据位置内的赋值在某些情况下仅进行浅拷贝。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0;

contract C {
    uint[] x;

    function f(uint[] memory memoryArr) public {
        // 深拷贝
        x = memoryArray;
        // 浅拷贝
        uint[] storage y = x;
        // 返回第八个元素
        y[7];
        // 通过y修改x
        y.pop();
        // 传递一个引用
        g(x);
        // 深拷贝
        h(x);
    }
    function g(uint[] storage) internal pure {}
    function h(uint[] memory) public pure {}
}
```

### 数组

数组类型的长度可以是动态的，也可以是编译时定长的。
Solidity对多维数组的解释方式与其他语言有些不同，例如`uint[][5]`表示一个有5个动态长度数组的数组。

#### 定长字节数组

`bytes1`到`bytes32`表示从长度为1到长度为32的一段字节。成员变量为`length`。

#### 动态长度数组

- `bytes`: 动态长度的字节数组。
- `string`: 动态长度的UTF-8编码的字符串。

字符串可以隐式转化成`bytes`，例如`bytes32 samevar = "stringliteral"`，字符串被解释为原始的字节并赋值给`samevar`。

可以通过使用keccak256比较两端字符串`keccak256(abi.encodePacked(s1)) == keccak256(abi.encodePacked(s2))`。
拼接字符串可以使用`string.concat(s1, s2)`。如果调用不带参数的`string.concat`或`bytes.concat`，它们将返回一个空数组。

数组的基本类型是列表中第一个表达式的类型，这样所有其他表达式都可以隐式转换为它。如果这不可能，则为类型错误。例如下面的例子中，`[1,2,3]`是`uint8[] memory`。如果想要`uint[] memory`则需要对第一个元素进行类型转化。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.16 <0.9.0;

contract C {
    function f() public pure {
        g([uint(1), 2, 3]);
    }
    function g(uint[3] memory) public pure {
        // ...
    }
}
```

定长数组不能被复制到动态长度的数组。如果要初始化动态大小的数组，则必须分配各个元素。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.16 <0.9.0;

contract C {
    function f() public pure {
        uint[] memory x = new uint[](3);
        x[0] = 1;
        x[1] = 3;
        x[2] = 4;
    }
}
```

#### 数组的成员

- `length`
- `push()`: 例子：`x.push().t = 2` 或 `x.push() = 2`
- `push(x)`
- `pop()`: 注意`pop`的gas开销依据要移除的元素的大小决定，如果要移除的元素是一个数组则开销较大。

#### `storage`类型数组的空引用

当对数组元素的引用存储在局部变量中，然后将之pop出之后，会出现空引用的情况。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.8.0 <0.9.0;

contract C {
    uint[][] s;

    function f() public {
        // 存储s的最后一个元素
        uint[] storage ptr = s[s.length - 1];
        // 移除s的最后一个元素
        s.pop();
        // 向不存在的数组写入数据
        ptr.push(0x42);
        // 向s中添加一个新元素不会添加一个新的空数组元素而是一个长度为1，元素为0x42的数组
        s.push();
        assert(s[s.length - 1][0] == 0x42);
    }
}
```

### 结构体

传递结构体参数时需要用方括号包裹，且如果有地址时地址需要使用双引号包裹。
如果传递结构体数组时需要在外层再次用方括号包裹。

```solidity
enum TokenType {
    ERC20,
    ERC721,
    ERC1155
}

struct Token {
    TokenType tokenType;
    address tokenAddr;
    uint256 tokenAmounts;
    uint256 tokenId;
    uint256 tokenPrice;
}

struct GeneralInfo {
    address loanToken; //token to borrow
    uint256 ltv; //8000 means 80%
    bool featuredFlag; //true, false
    uint256 loanAmount; //amounts to borrow
    uint256 interestRate; //1100 means 11%
    uint256 collateralThreshold; //8000 means 80%
    uint256 repaymentDate; //timestamp + 30days
    uint256 offerAvailable; //timestamp + 7days
}
```

正确的传参方式：

```
tokens:     [[0, '0x749B1c911170A5aFEb68d4B278cD5405C718fc7F',1000,0,0]],
general: ["0x749B1c911170A5aFEb68d4B278cD5405C718fc7F", 8000, false, 1000, 1100, 8000, 10000, 10000]
```

一个结构体应用的例子：

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.0 <0.9.0;

struct Funder {
    address addr;
    uint amount;
}

contract CrowdFunding {
    struct Campaign {
        address payable beneficiary;
        uint fundingGoal;
        uint numFunders;
        uint amount;
        mapping(uint => Funder) funders;
    }

    uint numCampaigns;
    mapping(uint => Campaign) campaigns;

    function newCampaign(address payable beneficiary, uint goal) public returns (uint campaignID) {
        campaignID = numCampaigns++; 
        Campaign storage c = campaigns[campaignID];
        c.beneficiary = beneficiary;
        c.fundingGoal = goal;
    }

    function contribute(uint campaignID) public payable {
        Campaign storage c = campaigns[campaignID];
        c.funders[c.numFunders++] = Funder({addr: msg.sender, amount: msg.value});
        c.amount += msg.value;
    }

    function checkGoalReached(uint campaignID) public returns (bool reached) {
        Campaign storage c = campaigns[campaignID];
        if (c.amount < c.fundingGoal)
            return false;
        uint amount = c.amount;
        c.amount = 0;
        c.beneficiary.transfer(amount);
        return true;
    }
}
```

结构不可能包含其自身类型的成员，但是可以包含自身类型的动态长度数组。

## 映射类型

映射类型的基本结构是`mapping(KeyType KeytName? => ValueType ValueName?)`。结构体、映射类型、数组不可以用作key。
映射类型的key不被保存在映射中，只有它的keccak256的hash值被保存。因此映射不存在长度或者键或值的集合的概念。
映射类型只能为`storage`类型，不可以被用作public类型的合约函数的返回值或参数，但是可以用作library的函数参数。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

contract MappingExample {
    mapping(address => uint) public balances;

    function update(uint newBalance) public {
        balances[msg.sender] = newBalance;
    }
}

contract MappingUser {
    function f() public returns (uint) {
        MappingExample m = new MappingExample();
        m.update(100);
        return m.balances(address(this));
    }
}
```

## 全局变量

只记录比较常用的全局变量。

### `block` 相关

- `blockhash(uint blockNumber) returns (bytes32)`: 既定块的哈希值
- `block.coinbas`L: `address payable`，当前矿工的地址。
- `block.gaslimit`: 当前块的gas限制
- `block.timestamp`: `uint`，当前块的时间戳。
- `gasleft() returns (uint256)`: 剩余的gas

### `msg` 相关

- `msg.data`: `bytes calldata`，完整的calldata
- `msg.sender`: `address`，消息的发送方
- `msg.sig`: `bytes4`，calldata的前四个字节，例如函数的标识
- `msg.value`: 随消息发送的以太币，单位为wei

msg 的所有成员的值，包括 msg.sender 和 msg.value 可以在每次外部函数调用时改变。

### `tx` 相关

- `tx.origin`: `address`，交易的发送者。

### `ABI`的编码解码函数

- `abi.decode(bytes memory encodeData, (...)) returns (...)`: 解码数据，第二个参数用于设定解码的数据的数据类型。例如`(uint a, uint[2] memory b, bytes memory c) = abi.decode(data, (uint, uint[2], bytes))`
- `abi.encode(...) returns (bytes memory)`: ABI-encode
- `abi.encodePacked(...) returns (bytes memory)`
- `abi.encodeWithSelector(bytes4 selector, ...) returns (bytes memory)`
- `abi.encodeWithSignature(string memory signature, ...) returns (bytes memory)`: 等价于`abi.encodeWithSelector(bytes4(keccak256(bytes(signature))), ...)`
- `abi.encodeCall(function functionPointer, (...)) returns (bytes memory)`: 用于外部合约的函数调用

### 错误处理相关

- `assert(boo condition)`: 用于处理内部错误，不满足会回滚
- `require(bool condition, string memory message?)`: 用于处理输入或外部组件错误，不满足会回滚，消耗的gas不回退，剩余的gas退回
- revert(string memory reason?): 触发回滚，用于比较复杂的校验情况
- revert Error(...): 可以节约gas，错误消息的长度会影响gas消耗数量

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.4;

contract VendingMachine {
    address owner;
    error Unauthorized();
    // error可以加参数
    error InsufficientFund(uint balance, uint withdrawAmount);
    function buy(uint amount) public payable {
        if (amount > msg.value / 2 ether)
            revert("Not enough Ether provided.");
        // Alternative way to do it:
        require(
            amount <= msg.value / 2 ether,
            "Not enough Ether provided."
        );
        // Perform the purchase.
    }
    function withdraw() public {
        if (msg.sender != owner)
            revert Unauthorized();
        // 带参数error
        revert InsufficientFund({balance: bal, withdrawAmount: _withdrawAmount});

        payable(msg.sender).transfer(address(this).balance);
    }
}
```

### 哈希

- `keccak256(bytes memory) returns (bytes32)`

### 合约相关

- `this`: 当前合约，可以显式地转化成`address`
- `super`: 合约的父合约

## delete操作符

- delete操作符可以用于将除map外的任何变量置为默认值
- 如果对动态数组使用delete则删除所有元素并置长度为0
- 如果对静态数组使用delete则重置所有索引值，数组长度不变
- 如果对map类型中的一个键使用delete，则会删除与该键相关的值

