---
title: Hardhat笔记
date: 2023-05-27 09:38:31
tags:
- hardhat
---

## Overview

Hardhat 是一个以太坊应用的开发脚手架。当使用Hardhat时HardhatRunner是交互的主要组件，可以协助管理和自动化开发智能合约时的重复性任务。

Hardhat围绕tasks和plugins构建。每次从命令行执行hardhat都是在执行一个task，例如 `npx hardhat compile` 为执行一个compile task。task之间可以相互调用。

## Quick Start

### 部署脚本

```ts
// deploy.ts
import {ethers} from "hardhat";

async function main() {
    const Box = await ethers.getContractFactory("Box");
    const box = await Box.deploy();
    await box.deployed();
}
```

### 命令行交互

```shell
npx hardhat console --network localhost
Welcome to Node.js v12.22.1.
Type ".help" for more information.
> const Box = await ethers.getContractFactory('Box');
undefined
> const box = await Box.attach('0x5FbDB2315678afecb367f032d93F642f64180aa3')
undefined
> await box.store(42)
{
  hash: '0x3d86c5c2c8a9f31bedb5859efa22d2d39a5ea049255628727207bc2856cce0d3',
...

```

### 通过代码交互

```ts
async function main() {
    const accounts = await ethers.provider.listAccounts();
    console.log(accounts);
    const contractAddr = "0x5fbdb2315678afecb367f032d93f642f64180aa3";
    const Box = await ethers.getContractFactory("Box");
    const box = await Box.attach(contractAddr);
    console.log((await box.retrieve()).toString());
    await box.store(114512);
    console.log((await box.retrieve()).toString());
}

```

## Configuration

```typescript
module.exports = {
 	defaultNetwork: "sepolia",
    networks: {
        hardhat: {
            
        },
        sepolia: {
            url: "https://sepolia.infura.in/v3/<key>",
            accounts: [privateKey1, privateKey2, ...]
        }
    },
    solidity: {
        version: "solidity-version",
        settings: {
            optimizer: {
                enabled: true,
                runs: 200
            }
        }
    },
    path: {
        sources: "./contracts",
   	 	tests: "./test",
    	cache: "./cache",
    	artifacts: "./artifacts"
    },
    mocha: {
        timeout: 40000
    }
}
```

### Networks configuation

`networks`用于将网络名称映射到对应的配置。Hardhat中有两种网络：基于JSON_RPC的网络与内置的Hardhat网络。通过配置 `defaultNetwork` 修改默认网络，默认为 `hardhat`

### `hardhat.config.ts`

hardhat的大多数功能来源于插件，所以需要在 `hardhat.config.ts` 中引入插件。

```typescript
import "@nomicfundation/hardhat-toolbox";

export default {
    solidity: "0.8.9"
};
```



## 测试单元

#### 测试环境

使用 `chai` 进行模拟区块链以及测试

```shell
npm install --save-dev chai
```

使用test-helpers进行更详尽的测试

```shell
$ npm install --save-dev @openzeppelin/test-helpers
```

#### 编写测试模块

编写测试模块比较好的实践是在`test/`下对 `contract/`中的每个合约建立一个测试文件。

```ts
// test/Box.test.js
// Load dependencies
import {ethers} from "hardhat";

import { expect } from 'chai';

// Start test block
describe('Box', function () {
    before(async function () {
        this.Box = await ethers.getContractFactory('Box');
    });

    beforeEach(async function () {
        this.box = await this.Box.deploy();
        await this.box.deployed();
    });

    // Test case
    it('retrieve returns a value previously stored', async function () {
        // Store a value
        await this.box.store(42);

        // Test if the returned value is the same one
        // Note that we need to use strings to compare the 256 bit integers
        expect((await this.box.retrieve()).toString()).to.equal('42');
    });
});
```

执行 `npx hardhat test` 会运行 `test/` 目录下的所有测试模块.

### 样例

#### 基础测试

```typescript
import {expect} from "chai";
import hre from "hardhat";
import {time} from "@nomicfoundation/hardhat-network-helpers";

describe('Lock', () => { 
    it("unlock time should be in the future", async function() {
        const lockedAmount = 1_000_000_000;
        const ONE_YEAR_IN_SECS = 356 * 24 * 60 * 60;
        const unlockTime = (await time.latest()) + ONE_YEAR_IN_SECS;
        const Lock = await hre.ethers.getContractFactory("Lock");
        const lock = await Lock.deploy(unlockTime, {
            value: lockedAmount
        });
        expect(await lock.unlockTime()).to.equal(unlockTime);
    });
 });
```

- `expect`：用于编写断言
- `hre`: hardhat runtime environment
- `hardhat-network-helper`: 与hardhat网络交互
- `describe`: `Mocha`的全局变量，用于描述和对测试分组
- `it`: `Mocha`的全局变量，用于描述和对测试分组

测试本体位于`it`函数中。

在上述测试模块中，`lockedAmount`是用于在合约中锁住的金额(单位为wei)，`unlockTime`是解锁的时间。`time.latest`是一个返回最新挖到的块的时间戳。使用 `hre.ethers.getContractFactory`获取Lock合约，然后将合约部署，同时将`unlockTime`作为参数传递给它的构造函数，并向之传递交易的一些参数，包括通过 `value` 表示的发送给合约的以太币。最后通过 `unlockTime()`获取合约内部的`unlockTime`并使用 `to.equal` 与上述设定的`unlockTime`对比。

其中所有的函数返回值均为Promise。

#### 测试会回滚的函数

```solidity
function withdraw() public {
  require(block.timestamp >= unlockTime, "You can't withdraw yet");
  require(msg.sender == owner, "You aren't the owner");
```

上述合约函数中存在回滚的条件。

```typescript
    it("Should revert with the right error if called to soon", async function() {
        const { lock, unlockTime } = await deployContract();
        await expect(lock.withdraw()).to.be.revertedWith("You can't withdraw yet");
    });
```

使用 `to.be.revertedWith` 断言交易发生回滚，原因在`revertedWith` 中给出。

其中 `.to.be.revertedWith` 是 hardhat Chai Matchers插件中的一部分。

上述两个测试中，断言中的await位置不相同，原因是第一个目的是比较两个值是否相同，内部的await用于等待值的获取，第二个的整个断言都是异步的因为必须等待交易被挖完矿，这意味着expect返回了一个promise。

#### 使用不同的地址

默认情况下部署和函数调用会使用配置的第一个账户。

```typescript
it("Should revert with the right error if called from another account", async function () {
  // ...deploy the contract...

  const [owner, otherAccount] = await ethers.getSigners();

  // we increase the time of the chain to pass the first check
  await time.increaseTo(unlockTime);

  // We use lock.connect() to send a transaction from another account
  await expect(lock.connect(otherAccount).withdraw()).to.be.revertedWith(
    "You aren't the owner"
  );
});
```

`hre.ethers.getSigners()`返回了一个数组，包含所有配置过的账户。使用 `lock.connect`方法连接到不同的账户。

#### Using fixtures

使用`beforeEach`钩子处理重复逻辑

```typescript
import { Contract } from "ethers";

describe("Lock", function () {
  let lock: Contract;
  let unlockTime: number;
  let lockedAmount = 1_000_000_000;

  beforeEach(async function () {
    const ONE_YEAR_IN_SECS = 365 * 24 * 60 * 60;
    unlockTime = (await helpers.time.latest()) + ONE_YEAR_IN_SECS;

    const Lock = await ethers.getContractFactory("Lock");
    lock = await Lock.deploy(unlockTime, { value: lockedAmount });
  });

  it("some test", async function () {
    // use the deployed contract
  });
});
```

使用 `helpers.loadFixture` 解决多个测试之间共享变量的问题。第一次执行`loadFixture`，fixture函数被执行。当第二次执行，`loadFixture` 会重置网络的状态到fixture执行后的状态而不是再次执行fixture，从而撤销过去测试修改的状态。

```typescript
describe('Lock', () => { 
    async function deployOneYearLockFixture() {
        const lockedAmount = 1_000_000_000;
        const ONE_YEAR_IN_SECS = 365 * 24 * 60 * 60;
        const unlockTime = (await helpers.time.latest()) + ONE_YEAR_IN_SECS;
        const Lock = await hre.ethers.getContractFactory("Lock");
        const lock = await Lock.deploy(unlockTime, {
            value: lockedAmount
        });
        return {
            lock,
            unlockTime,
            lockedAmount
        }
    }

    it("Sould set the right unlockTime", async function() {
        const { lock, unlockTime } = await helpers.loadFixture(deployOneYearLockFixture);

        // assert that the value is correct
        expect(await lock.unlockTime()).to.equal(unlockTime);
    });
});
```



## 常用命令

`npx hardhat node`

> 开启本地测试网，默认端口是8545
>
> 同时生成具有ether的账号，这些账号是在本地测试网开启时生成的，故每次开启的账号都会不同。

`npx hardhat run --network localhost <deploy-script-path>`

> 将合约部署在本地测试网

`npx hardhat console --network localhost`

> 开启与本地测试网交互的命令行

`npx hardhat coverage`

> 测试在项目中的覆盖度