---
title: Solidity合约的基本结构
date: 2023-05-16 14:42:07
tags:
- Solidity
- Smart Contract
---
## 合约的基本结构

- 状态变量(State Variables)

永久保存于合约中，且会随合约储存在区块链上，因此需要考虑去除不必要的数据存储以减少部署时的开销。
以 `Storage` 类型存储。
同时每一个状态变量会隐含创建一对Getter与Setter，在使用外部调用call时会使用到。

- 函数(Functions)

基本结构时 `function <functionName>(<parameter type, <parameter name>) [public|private|internal|external] [pure|view|payable|省略] [virtual] [override] [returns (<return types>)]`。其中，装饰器的位置不固定。
函数可以定义于合约内部，也可以定义于合约外部。

- 函数装饰器(Function Modifiers)

装饰器可以被继承、重写与传参。

- 事件(Events)

- Errors

Errors可以用于 `revert` 语句中。

- 结构体

- 枚举类型

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0;

contract SimpleContract {
    // State variables
    uint storedData;

    // Functions
    function bid() public pure {
        emit HighestBidIncreased(msg.sender, uint storedData);
        revert NotEnoughFunds(...);
    }

    // Events
    event HighestBidIncreased(address bidder, uint amount);

    // Errors
    error NotEnoughFunds(uint requested, uint available);

    // Struct Types
    struct Voter { // Struct
        uint weight;
        bool voted;
        address delegate;
        uint vote;
    }
}
```

## 合约的源文件布局

### SPDX License Identifier

开源的合约可以得到用户对合约信赖，所以鼓励对合约进行开源，但是开源会牵扯到版权等问题，所以需要使用 `// SPDX-License-Identifier: License Name` 在文件首部说明使用的开源协议。

#### 一些常用的开源协议

{% asset_img 开源协议.drawio.png 常用的开源协议 %}

### Pragmas

`pragma` 字段用于确定编译器的一些特征。

- solidity的版本: `pragma solidity ...;`

- ABI Coder的版本: 

  - `pragma abicoder v2`: default

  - `pragma abicoder v1`

### Import

常用的格式如下：

```solidity
/**
从目标文件中引入所有全局符号。
注意容易造成符号名污染的问题。
**/
import "fileName";

import * as symbolName from "fileName";

import "filename" as symbolName;

/**
解构
**/
import {symbol1 as alias, symbol2} from "filename";
```