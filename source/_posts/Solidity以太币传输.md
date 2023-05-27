---
title: Solidity以太币传输
date: 2023-05-17 09:57:43
tags:
- Solidity
- Smart Contract
---

## 发送方式

- `transfer()`：消耗2300gas，失败会抛出异常
- `send()`：消耗2300gas，返回表示成功状态的bool值，如果使用必须要校验是否执行成功
- `call()`: 传递交易剩余的gas或设置的gas，不限定2300gas，返回一个bool表明调用状态。
  - 通过调用fallback(下方例子，推荐)
  - 通过调用receive(下方例子，推荐)

transfer与call使用2300gas以防止重入攻击，但公链升级后可能导致 gas 不足。所以推荐使用 call() 函数，但需做好防重入攻击准备。

只有当msg.data为空且目标合约具有receive函数才可以使用receive的方式进行转账，其他则为fallback方式。

## 接收

想接受ether的合约至少包含以下方法的一个：

- `receive() external payable`: msg.data为空时调用
- `fallback() external payable`: msg.data非空时调用，为执行default逻辑而生，**顺便支持接收ether**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

contract ReceiveEther {
    /*
    Which function is called, fallback() or receive()?

                sender ether
                    |
             msg.data is empty?
                /       \
            yes          no
             /             \
      receive() exist?     fallback()
          /    \
        yes     no
       /          \
  receive()     fallback()
  */

    string public message;

    // Function to receive Ether. msg.data must be empty
    receive() external payable {
        message = "receive called!";
    }

    // Fallback function is called when msg.data is not empty
    fallback() external payable {
        message = "fallback called!";
    }

    function getBalance() public view returns (uint) {
        return address(this).balance;
    }

    function setMsg(string memory _msg) public {
        message = _msg;
    }
}

contract SendEther {
    function sendViaTransfer(address payable _to) public payable {
        // This function is no longer recommended for sending Ether. (不建议使用)
        _to.transfer(msg.value);
    }

    function sendViaSend(address payable _to) public payable {
        // Send returns a boolean value indicating success or failure.
        // This function is not recommended for sending Ether. (不建议使用)
        bool sent = _to.send(msg.value);
        require(sent, "Failed to send Ether");
    }

    function sendViaCallFallback(address payable _to) public payable {
        // Call returns a boolean value indicating success or failure.
        // This is the current recommended method to use. (推荐使用)
        (bool sent, bytes memory data) = _to.call{value: msg.value}(abi.encodeWithSignature("noExistFuncTest()"));
        require(sent, "Failed to send Ether");
    }

    function sendViaCallReceive(address payable _to) public payable {
        // Call returns a boolean value indicating success or failure.
        // This is the current recommended method to use.(推荐使用)
        (bool sent, bytes memory data) = _to.call{value: msg.value}("");
        require(sent, "Failed to send Ether");
    }
}
```