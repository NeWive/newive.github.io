---
title: Solidity智能合约基本概念
date: 2023-05-17 14:30:56
tags:
- Smart Contract
- NFT
---

## 概述

智能合约（smart contract）的定义如下：

智能合约是一种旨在以信息化方式传播、验证或执行合同的计算机协议，它允许在没有第三方的情况下进行可信交易。这些交易可追踪且不可逆转。

以太坊中最大特点是引入了智能合约，智能合约运行平台为脚本提供了更加丰富的表现力，为区块链的应用场景增加了无限的可能性。

## 以太坊账户

- 外部账户(EOA): 普通用户持有的账户，它的地址由公钥确定，通过公私钥对生成。
- 合约账户(CA): 合约账户中除了存储代码（编译后的字节码），其他功能与外部账户完全一样。

合约账户由外部账户调用，之后合约账户也可继续调用合约账户。也就是合约账户最初始只能是外部账户来调用。

## gas

在以太坊中引入了汽油费（Gas）的概念，引入的目的是执行智能合约中可能消耗的资源会很大，所以需要收取交易发起人的汽油费。

GasLimit为最大汽油费，当一个全节点收到对智能合约的调用时会先按照最大汽油费一次性从账户扣除掉，再根据实际情况，若最后合约执行完毕有多余的汽油费将退回，若汽油费不够，智能合约将状态回滚，并且汽油费不会退回。

## NFT 概述

一个token是区块链中某个事物的代表。通过将这些事物以token作为代表，可以允许其在智能合约中流通。

一个token合约本质是一个以太坊智能合约，发送token即为调用合约中的一个方法。

NFT(Non-fungible tokens)是用于提供所有权证明的加密的独一无二的代币。

NFT是包含有记录于*智能合同(smart contract)*中的身份识别信息的数字资源。记录于智能合同中的信息是对NFT唯一性的保证。其中不可替代性(Non-fungible)即代币不可被直接替换为另一个代币，也不能将两个NFT交换，因为两个NFT之间不具有相似性。

比特币是一种可替换的代币，比特币也可以划分为更小单位的比特币的集合，类比不同面值的人民币。因此*可替代的货币是可分割的，不可替代的货币是不可分割的*。

OpenSea、Rarible是NFT领域主要的交易渠道。

NFT和对应的智能合同允许包含更详细的属性，比如拥有者的身份、丰富的元数据或重要的文件链接。

NFT、相应的协议和智能合约依然处于开发状态。搭建用于管理和创建NFT的应用和平台依旧相对复杂。

NFT为安全令牌的创建和数字以及现实世界的资源的代币化提供了可能。实物的资源比如财产可以被标记为部分或共享所有权。如果这些安全令牌是不可替代的，那这些资源的所有权完全可追溯，即使代币代表的部分拥有权被售卖。

NFT在IP Rights(Copyrights、Trademarks、Patents、TradeSecrets)中一般指Copyrights。

NFT中的token绑定了唯一一个uri，也就是metadata的链接。在metadata中会进一步包含一个资源的URI，这个url可以是图片都，也可以是video的。从技术层面来 讲，这是一个非常普通的token，代码非常简单，NFT是建立在区块链之上的，是智能合约成就了NFT，而不是NFT本身有多么复杂。

## 智能合约运行流程

1.编写智能合约代码，并编译成字节码。

2.部署智能合约。过程是向“0”地址发送一笔带有智能合约字节码数据的交易，这个交易会生成该智能合约的地址，并将字节码存储在该地址下的状态树中。

3.执行智能合约（调用智能合约函数）。向智能合约地址发送一个交易，该交易携带被调用的智能合约函数信息及调用参数，携带的信息遵循ABI编码协议。

4.智能合约地址收到这样的调用合约函数的交易，首先会解码数据，根据结果查找到对应函数的入口，再传入参数执行该函数。

5.执行函数的过程是状态转换的过程，执行完成后会扣除调用者相应的Gas花费。

6.状态转换的过程会全网同步并被再次执行验证，确保执行结果一致，这样通过验证后的交易会记录到区块中，同时更新状态数据。

## NFT Metadata

为了让 OpenSea 为 ERC721 代币提取链下元数据，合约需要返回一个指向托管元数据的 URI。为了找到这个 URI，OpenSea、Rarible 和其他流行的市场将使用 ERC721URIStorage 标准中包含的 tokenURI 方法。

ERC721 中的 tokenURI 函数应该返回一个 HTTP 或 IPFS URL，例如 ipfs://bafkreig4rdq3nvyg2yra5x363gdo4xtbcfjlhshw63we7vtlldyyvwagbq。查询时，此 URL 应返回一个 JSON 数据块，其中包含token的元数据。

根据OpenSea的文档，NFT Metadata应该按照如下格式存储
```json
{ 
  "description": "YOUR DESCRIPTION",
  "external_url": "YOUR URL",
  "image": "IMAGE URL",
  "name": "TITLE", 
  "attributes": [
    {
      "trait_type": "Base", 
      "value": "Starfish"
    }, 
    {
      "trait_type": "Eyes", 
      "value": "Big"
    }, 
    {
      "trait_type": "Mouth", 
      "value": "Surprised"
    }, 
    {
      "trait_type": "Level", 
      "value": 5
    }, 
    {
      "trait_type": "Stamina", 
      "value": 1.4
    }, 
    {
      "trait_type": "Personality", 
      "value": "Sad"
    }, 
    {
      "display_type": "boost_number", 
      "trait_type": "Aqua Power", 
      "value": 40
    }, 
    {
      "display_type": "boost_percentage", 
      "trait_type": "Stamina Increase", 
      "value": 10
    }, 
    {
      "display_type": "number", 
      "trait_type": "Generation", 
      "value": 2
    }]
  }
```