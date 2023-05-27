---
title: Solidity合约安全
date: 2023-05-17 10:13:49
tags:
- Solidity
- Smart Contract
---

## 重入

合约 (A) 与另一个合约 (B) 的任何交互以及以太币的任何转移都会将控制权移交给该合约 (B)。这使得 B 可以在此交互完成之前回调 A。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.0 <0.9.0;

// 包含bug的合约
contract Fund {
    mapping(address => uint) shares;
    
    function withdraw() public {
        if (payable(msg.sender).send(shares[msg.sender]))
            shares[msg.sender] = 0;
    }
}
```
Ether 传输总是可以包括代码执行，因此接收者可以是一个调用撤回的合约。

上述代码造成的危害较小，因为send对gas消耗的限制。但如果使用call则会使攻击者获取多次退款，因为call默认会转发所有gas。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.2 <0.9.0;

contract Fund {
    mapping(address => uint) shares;

    function withdraw() public {
        (bool success,) = msg.sender.call{value: shares[msg.sender]}("");
        if (success)
            shares[msg.sender] = 0;
    }
}
```

上述代码中攻击者可以回调发送合同的函数，因而可以多次调用withdraw而在call未执行完时shares不会更新，从而实现hack的目的。

### 解决方案：

- 优先更新shares

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.0 <0.9.0;

contract Fund {

    mapping(address => uint) shares;

    function withdraw() public {
        uint share = shares[msg.sender];
        shares[msg.sender] = 0;
        payable(msg.sender).transfer(share);
    }
}
```

- 将withdraw的过程整合为一个原子操作，加锁

```solidity
contract Locker {
    bool private locked = false;

    modifier lock() {
        require(
            !locked,
            "Reentrant call."
        );
        locked = true;
        _;
        locked = false;
    }
}

contract Fund is Locker {

    mapping(address => uint) shares;

    function withdraw() public lock {
        uint share = shares[msg.sender];
        shares[msg.sender] = 0;
        payable(msg.sender).transfer(share);
    }
}
```

但是上锁之后不会对后续请求进行排队。

## 循环与gas限制

如果循环的结束条件是根据变量决定的，则循环次数不确定，有可能导致消耗过量的gas。

## 发送和接收ether

如果一个合约在没有函数被调用的情况下接收ether，则receive或者fallback函数将被调用。如果合约中没有这两个函数的实现则ether将被拒绝。

在调用上述两个函数的过程中，合约只能依靠当时可用的“gas 津贴”（2300 gas）。这个津贴不足以修改storage，为了确保能够在上述情况下顺利接受ether需要确保receive与fallback对gas的要求。

尽可能使用最精确的单位来表示 Wei 数量，因为会丢失任何因缺乏精度而四舍五入的单位。

### 使用transfer时需要注意的内容

- 如果接收方是个合约则会调用它的receive或fallback函数，从而接收方可以回调发送方的函数
- 发送ether会因为调用栈深度超过1024导致失败。可以考虑使用`send`避免这个问题并检查`send`的返回值。更好的处理方式时为自身添加一个可以退款的方式
  
## 返回值校验

调用外部合约函数时，有些函数调用失败不会抛出错误回滚交易而是返回 false，如果忘记检查函数返回值会导致误以为调用成功。

例如：approve等方法就是不安全的，因此引入了safeApprove

当使用call方法调用approve时，如果没有校验返回值，则即使失败，也不会抛出异常。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.8;

contract ReturnValue {
  function callchecked(address callee) public {
    (bool success, ) = callee.call("");
    require(success, "call failed!");
  }

  function callnotchecked(address callee) public {
    // 未校验
    (bool success, ) = callee.call("");
  }
}
```

## 合约自杀式攻击

合约自杀时，会将合约自身持有的ether全部转入到指定地址之中。

如果某个合约使用了balance方法来进行校验时，有可能会出现攻击漏洞。

### 案例：

规则如下：
- 游戏的目标是成为第 7 个存入 1 个以太币的玩家。
- 玩家一次只能存入 1 个以太币。
- 获胜者将能够提取所有以太币。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

contract EtherGame {
    uint public targetAmount = 7 ether;
    address public winner;

    function deposit() public payable {
        require(msg.value,  "You can only send 1 ether");

        uint balance = address(this).balance;
        require(balance <= targetAmount, "Game Over);

        if(balance == targetAmount) {
            winner = msg.sender;
        }
    }

    function claimReward() public {
        require(msg.sender == winner, "Not Winner");
        (bool sent, ) = msg.sender.call{value: address(this).balance}("");
        require(sent, "Failed to send ether");
    }
}

contract Attack {
    EtherGame etherGame;

    constructor(EtherGame _etherGame) {
        etherGame = EtherGame(_etherGame);
    }

    function attack() public payable {
        // 不经过deposit使劲发送以太币使超过7个

        //将address转化为payable
        address payable addr = payable(address(etherGame));
        selfdestruct(addr);
    }
}
```

过程：

1.部署EtherGame 
2. 玩家（比如爱丽丝和鲍勃）决定玩游戏，每人存入 1 个以太币。 
3. 使用 EtherGame 地址部署攻击 
4. 调用 Attack.attack 发送 5 个以太币。这会破坏游戏 没有人能成为赢家。

解决方案：

不使用balance进行校验。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

contract EtherGame {
    uint public targetAmount = 3 ether;
    uint public balance;
    address public winner;

    function deposit() public payable {
        require(msg.value == 1 ether, "You can only send 1 Ether");

          // ！！通过状态变量来统计ether，此时即使合约自杀传入ether，也不会干扰条件判断逻辑！！
        balance += msg.value;

        require(balance <= targetAmount, "Game is over");

        if (balance == targetAmount) {
            winner = msg.sender;
        }
    }

    function claimReward() public {
        require(msg.sender == winner, "Not winner");

        (bool sent, ) = msg.sender.call{value: balance}("");
        require(sent, "Failed to send Ether");
    }
}
```

## 访问私有数据private data

所有链上的数据都可以被访问；不要把任何敏感信息存储在链上

private修饰符表示链上合约无法访问该数据，但是可以通过链下（读取slot的方式读取私有数据）

// TODO: 未看代码验证

## `tx.origin` 攻击

不要使用`tx.origin`进行鉴权。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.7.0 <0.9.0;

contract TxUserWallet {
    address owner;

    constructor() {
        owner = msg.sender;
    }

    function transferTo(address payable dest, uint amount) public {
        require(tx.origin == owner);
        dest.transfer(amount);
    }
}

//现在有人欺骗你将以太币发送到这个攻击钱包的地址：

interface TxUserWallet {
    function transferTo(address payable dest, uint amount) external;
}

contract TxAttackWallet {
    address payable owner;

    constructor() {
        owner = payable(msg.sender);
    }

    receive() external payable {
        TxUserWallet(msg.sender).transferTo(owner, msg.sender.balance);
    }
}
```
如果钱包检查了`msg.sender`以获得授权，它将获得攻击钱包的地址，而不是所有者的地址。但是通过检查 tx.origin，它得到了启动交易的原始地址，这仍然是所有者的地址。

## 溢出

默认在checked模式下，溢出总是会回滚。如果无法避免溢出，则可能导致智能合约卡在某个状态。尝试使用`require`控制输入的范围。