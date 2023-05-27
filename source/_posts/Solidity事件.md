---
title: Solidity事件
date: 2023-05-17 09:59:15
tags:
- Solidity
- Smart Contract
---

事件是区块链上的日志，每当用户发起操作的时候，可以发送相应的事件，常用于：

1. 监听用户对合约的调用
2. 便宜的存储（用合约存储更加昂贵）

通过链下程序（如：subgraph）对合约进行事件监听，可以对Event进行搜集整理，从而做好数据统计，常用方式：

- 合约触发后发送事件
- subgraph对合约事件进行监听，计算（如：统计用户数量）
- 前端程序直接访问subgraph的服务，获得统计数据（这避免了在合约层面统计数据的费用，并且获取速度更快）

```solidity
pragma solidity ^0.8.13;

contract Event {
	event Log(address indexed sender, string message);
	event AnotherLog();
	event TestAnonymous(address indexed sender, uint256 num) anonymous; // 匿名事件
	function test() public {
		emit Log(msg.sender,  "Hello World");
		emit Log(msg.sender, "Hello EVM");
		emit AnotherLog();
	}
}
```

一个事件内可以最多将三个字段修饰为indexed，当使用indexed关键字时，更加方便索引，并且：

1. 如果修饰的是值类型的，则直接展示；
2. 如果是非值类型，如：array，string等，则使用keccak256哈希值。
3. indexed：方便索引
4. non-indexed：没有解码，需要使用abi解码后才知道内容，不加indexed是data

其他：

1. Log也在区块链账本中，和合约存储在不同的结构中；
2. Log是由交易执行产生的数据，它是不需要共识的，可以通过重新执行交易生成；
3. Log是经由链上校验的，无法造假，因为一笔交易的Receipt Hash是存在链上的（Header中）