---
title: Openzeppelin笔记
date: 2023-05-27 09:39:38
tags:
- openzeppelin
- solidity
---

## 通过继承的方式使用

```solidity
pragma solidity ^0.8.13;

import "@openzeppin/contracts/access/AccessControl.sol";

contract ModifierAccessControl is AccessControl {
	function revokeRole(bytes32 role, address account) public override {
		require(role != DEFEAULT_ADMIN_ROLE, "ModifierdAccessControl: Cannot revoke def");
		super.revokeRule(role, account);
	}
}
```

## 使用hooks

使用 `_beforeTokenTranser` hook拓展ERC20中的 `IERC721Receiver`

```solidity
import "@openzepplin/contracts/token/ERC20/ERC20.sol";

contract ERC20WithSafeTranser is ERC20 {
	function _beforeTOkenTransfer(address from, address to, uint256 amount) internal virtual override {
	super._beforeTokenTranser(from, to, amount);
	require(_validRecipient(to), "ERC20WithSafeTransfer: invalid recipient")
	}
}
```

Hook的规则：

1. 当重写hook时对新的函数添加 `virtual` 属性以便于后续子合约对其拓展。
2. 总是通过 `super` 的方式调用父hook以确保在继承树中的所有hook都被调用。

## upgradeability

可升级的合约：

```shell
npm install @openzeppelin/contracts-upgradeable
```

## 访问控制

### 合约的所有权

`Ownable.sol` 提供对判定合约函数所有者相关的API。

```solidity
pragma solidity ^0.8.13;

import "@openzepplin/contracts/access/Ownable.sol";

contract MyContract is Ownable {
	function normalThing() public {
		//...
	}
	function specialThing() public onlyOwner {
		
	}
}
```

`Ownable`合约默认部署合约者作为合约的所有者。同时该合约还可以

- `transferOwnership`: 修改合约所有人
- `renounceOwnership`: 当前所有人放弃所有权

注意移除所有者之后`onlyOwner`装饰器将不可用。

### 基于角色的访问控制

`AccessControl.sol` 提供基于角色的访问控制相关的API。

通过创建一个新的角色标识符 `role identifier` 来进行访问控制。一般 `role identifier` 是一段通过keccak256生成的哈希。

通过 `_grantRole` 创建角色。

```solidity
pragma solidity ^0.8.13;
contract MyToken is ERC20, AccessControl {
	bytes32 public constant MINTER_ROLE = keccak256("MINTER");
	constructor(address minter) ERC20("MyToken", "TKN") {
		_grantRole(MINTER_ROLE, minter);
	}
	
	function mint(address to, uint256 amount) public {
		require(hasRole(MINTER_ROLE, msg.sender), "Caller is not a minter");
		_mint(to, amount);
	}
}
```

使用 `onlyRole` 装饰器

```solidity
import "@openzepplin/contracts/access/AccessControl.sol";
import "@openzepplin/contracts/token/ERC20/ERC20.sol";

contract MyToken is ERC20, AccessControl {
	bytes32 public constant MINTER_ROLE = keccak256("MINTER");
	bytes32 public constant BURNER_ROLE = keccak256("BURNER");
	constructor(address minter, address burner) ERC20("MyToken", "TKN") {
		_grantRole(MINTER_ROLE, minter);
		_grantRole(BURNER_ROLE, burner);
	}
	
	function mint(address to, uint256 amount) public onlyRole(MINTER_ROLE) {
		_mint(to, amount);
	}
	
	function burn(address from, uint256 amount) public onlyRole(BURNER_ROLE) {
		_burn(from, amount);
	}
}
```

### 管理角色权限

默认情况下，具有角色的帐户不能授予它或从其他帐户中撤销它。为了动态的管理角色权限需要一个角色管理员(role admin)。

每个角色都有一个相关联的管理员角色用来授权调用 `_grantRole`与 `_revokeRole`。`AccessControl.sol` 具有一个特殊的role `DEFAULT_ADMIN_ROLE`以作为所有角色的默认管理员角色，可以通过调用 `_setRoleAdmin` 修改管理员角色。

但是 `DEFAULT_ADMIN_ROLE` 因为也是自己的管理员所以具有风险。可以使用 `AccessControlDefaultAdminRules` 来对管理员角色添加强制安全措施。

```solidity
pragma solidity ^0.8.13;

import "@openzepplin/contracts/access/AccessControl.sol";
import "@openzepplin/contracts/token/ERC20/ERC20.sol";

contract MyToken is ERC20, AccessControl {
	bytes32 public constant MINTER_ROLE = keccak256("MINTER");
	bytes32 public constant BURNER_ROLE = keccak256("BURNER");
	constructor() ERC20("MyToken", "TKN") {
		_grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
	}
	
	function mint(address to, uint256 amount) public onlyRole(MINTER_ROLE) {
        _mint(to, amount);
    }

    function burn(address from, uint256 amount) public onlyRole(BURNER_ROLE) {
        _burn(from, amount);
    }
}
```

由于DEFAULT_ADMIN_RULE被设置成为 `msg.sender`，该账户可以调用 `grantRole` 来授予别的账户以角色。

### 遍历特权角色

`AccessControl.sol` 使用 `EnumerableSet` 来保存角色信息，可以通过 `getRoleMemberCount` 来获取特权角色账户的数量。通过 `getRoleMember`来获取每一位角色。

```js
const minterCount = await myToken.getRoleMemberCount(MINTER_ROLE);

const members = [];
for(let i = 0; i < minterCount; ++i) {
    members.push(await myToken.getRoleMember(MINTER_ROLE, i));
}
```

## 延迟操作(Delayed Operation)

访问控制无法防止内部管理员搞事，因此可以使用 `TimelockController`。

`TimelockController` 是一个由提议者`proposers`和执行者`executors`管理的代理。当设置好管理员后它确保提议者下令进行的任何维护操作都可能会延迟。这个延迟为合约的用户提供时间确保维护操作的合理性。

默认情况下部署合约的地址为 `TimelockController` 在时间锁上的管理员。这个角色授予任命proposer、executor和其他管理员的权利。

`TimelockController` 需要至少任命一个proposer和一个executor。这些角色可以是相同的账户。同时角色 `ADMIN_ROLE`、  `PROPOSER_ROLE`、`EXECUTOR_ROLE` 基于`AccessControl` 实现。

## 合约的标准

- ERC20: 同质化资源的token标准
- ERC721: 非同质化token(NFT)的标准
- ERC777: 更丰富的同质化token标准

## ERC20

符合ERC20的token主要用于交换货币、投票选举等。

```solidity
import "@openzepplin/contract/ERC20/ERC20.sol";

contract GLDToken is ERC20 {
	constructor(uint256 initialSupply) ERC20("Gold", "GLD") {
		_mint(msg.sender, initialSupply);
	}
}
```

## ERC721

ERC721是代表NFT所有权的一个标准。

### 样例

使用ERC721处理游戏中的物品。当一个物品要奖励给一个玩家时，它会被进行挖矿获取token并发送给玩家。玩家可以自由的保管这些token或者是与他人进行交易。需要注意的是任何人都可以调用 `awardItem` 来挖掘物品，为了约束这一行为需要增加访问控制。

```solidity
pragma solidity ^0.8.13;

import "@openzepplin/contract/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzepplin/contracts/utils/Counters.sol";

contract GameItem is ERC721URIStorage {
	using Counters for Counters.Counter;
	Counters.Counter private _tokenIds;
	
	constructor() ERC721("GameItem", "ITM") {
	
	}
	
	function awardItem(address player, string memory tokenURI) {
		uint256 newItemId = _tokenIds.current();
		_mint(player, newItemId);
		_setTokenURI(newItemId, tokenURI);
		_tokenIds.increment();
		return newItemId;
	}
}
```

`ERC721URIStorage`合约是ERC721的一个实现，包含元数据标准扩展 `IERC721Metadata` 和一个关于每个令牌元数据的机制。使用 `_setTokenURI`保存一个物品的元数据。

需要注意的是ERC721的token不可再分割，且是独一无二的。

## 链上治理(On-Chain Governance)



治理协议通常以一个特殊目的的合约实现，称为 `Governor`。

### 样例

#### Setup

在治理协议的构建中每个账户的选举权利会由一个ERC20 Token决定。这个token必须实现`ERC20Votes`拓展。这个拓展会持续追踪历史余额以便投票权是从过去的快照而不是当前余额中检索的，这是一个对防止多次选举的重要保护。

```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Votes.sol";

contract MyToken is ERC20, ERC20Permit, ERC20Votes {
	constructor() ERC20("MyToken", "MTK") ERC20Permit("MyToken") {}
	
	function _afterTokenTransfer(address from, address to, uint256 amount) internal override(ERC20m ERC20Votes) {
		super._afterTokenTransfer(from, to, amount);
	}
	    function _mint(address to, uint256 amount)
        internal
        override(ERC20, ERC20Votes)
    {
        super._mint(to, amount);
    }

    function _burn(address account, uint256 amount)
        internal
        override(ERC20, ERC20Votes)
    {
        super._burn(account, amount);
    }
}
```

如果您的项目已经有一个不包含 `ERC20Votes` 且不可升级的实时令牌，您可以使用 `ERC20Wrapper` 将其包装在治理token中。这将允许token持有者通过一对一包装他们的token来参与治理。

```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Votes.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Wrapper.sol";

contract MyToken is ERC20, ERC20Permit, ERC20Votes, ERC20Wrapper {
    constructor(IERC20 wrappedToken)
        ERC20("MyToken", "MTK")
        ERC20Permit("MyToken")
        ERC20Wrapper(wrappedToken)
    {}

    // The functions below are overrides required by Solidity.

    function _afterTokenTransfer(address from, address to, uint256 amount)
        internal
        override(ERC20, ERC20Votes)
    {
        super._afterTokenTransfer(from, to, amount);
    }

    function _mint(address to, uint256 amount)
        internal
        override(ERC20, ERC20Votes)
    {
        super._mint(to, amount);
    }

    function _burn(address account, uint256 amount)
        internal
        override(ERC20, ERC20Votes)
    {
        super._burn(account, amount);
    }
}
```

> 注意：目前 OpenZeppelin 合约中唯一可用的其他投票权来源是 ERC721Votes。
>
> 不提供此功能的 ERC721 代币可以使用 ERC721Votes 和 ERC721Wrapper 的组合包装成投票代币。

#### Governor

考虑以下问题

- 如何决定选举权力

  > 使用 GovernorVotes模块，该模块连接到 IVotes 实例，以根据提案激活时账户持有的代币余额来确定账户的投票权。
  >
  > 这个模块要求一个地址的token作为构造函数的参数。

- 需要多少选举者才合法

  > 使用 GovernorVotesQuorumFraction模块，与ERC20Votes一起将人数定义为一个检索提案投票权的区块总供应量的百分比。

- 人在投票时有什么选项，这些票是如何被计数的

  > 使用 GovernorCountingSimple模块提供选项，包括 `For`, `Against`, `Abstain`。只有 `For`, `Abstain` 被计入人数。

- 用什么类型的token作为票

同时Governor还需要确定几个参数

> These parameters are specified in the unit defined in the token’s clock. Assuming the token uses block numbers, and assuming block time of around 12 seconds, we will have set votingDelay = 1 day = 7200 blocks, and votingPeriod = 1 week = 50400 blocks.

- votingDelay: 在创建提议后多长时间投票权确定

  > ex: votingDelay = 1 day = 7200 blocks

- votingPeriod: 提议的投票持续多长时间

  > ex: votingPeriod = 1 week = 50400 blocks.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.2;

import "@openzeppelin/contracts/governance/Governor.sol";
import "@openzeppelin/contracts/governance/compatibility/GovernorCompatibilityBravo.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorVotes.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorVotesQuorumFraction.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorTimelockControl.sol";

contract MyGovernor is Governor, GovernorCompatibilityBravo, GovernorVotes, GovernorVotesQuorumFraction, GovernorTimelockControl {
    constructor(IVotes _token, TimelockController _timelock)
        Governor("MyGovernor")
        GovernorVotes(_token)
        GovernorVotesQuorumFraction(4)
        GovernorTimelockControl(_timelock)
    {}

    function votingDelay() public pure override returns (uint256) {
        return 7200; // 1 day
    }

    function votingPeriod() public pure override returns (uint256) {
        return 50400; // 1 week
    }

    function proposalThreshold() public pure override returns (uint256) {
        return 0;
    }

    // The functions below are overrides required by Solidity.

    function state(uint256 proposalId)
        public
        view
        override(Governor, IGovernor, GovernorTimelockControl)
        returns (ProposalState)
    {
        return super.state(proposalId);
    }

    function propose(address[] memory targets, uint256[] memory values, bytes[] memory calldatas, string memory description)
        public
        override(Governor, GovernorCompatibilityBravo, IGovernor)
        returns (uint256)
    {
        return super.propose(targets, values, calldatas, description);
    }

    function _execute(uint256 proposalId, address[] memory targets, uint256[] memory values, bytes[] memory calldatas, bytes32 descriptionHash)
        internal
        override(Governor, GovernorTimelockControl)
    {
        super._execute(proposalId, targets, values, calldatas, descriptionHash);
    }

    function _cancel(address[] memory targets, uint256[] memory values, bytes[] memory calldatas, bytes32 descriptionHash)
        internal
        override(Governor, GovernorTimelockControl)
        returns (uint256)
    {
        return super._cancel(targets, values, calldatas, descriptionHash);
    }

    function _executor()
        internal
        view
        override(Governor, GovernorTimelockControl)
        returns (address)
    {
        return super._executor();
    }

    function supportsInterface(bytes4 interfaceId)
        public
        view
        override(Governor, IERC165, GovernorTimelockControl)
        returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }
}
```

#### Timelock

> 注意，当使用Timelock时，是Timelock执行决议，因此Timelock合约拥有资金、所有权和对角色访问控制权。

Timelock需要使用AccessControl设置角色。

- `Proposer`：控制排队操作，且必须唯一
- `Executor`：控制执行已经存在且可行的操作，可以将之设置为0地址以便任何人执行操作。
- `Admin`：授予和撤销上述两个角色，默认会将Timelock自身设置为Admin

## ERC 721 API

EIC制定了四个接口：

- IERC721: 在所有相关接口中包含的核心功能
- IERC721Metadata: 增加名称、标志和token URI的科学拓展，常用
- IERC721Enumerable: 允许遍历链上的token，因为消耗大量gas不常用
- IERC721Receiver: 如果想要通过 `safeTransferFrom` 接收token则必须实现的接口

- ERC721Consecutive
- ERC721URIStorage: 一种更灵活但是开销更大的存储元数据的方法
- ERC721Votes
- ERC721Royalty
- ERC721Pausable
- ERC721Burnable
- ERC721Wrapper: 用于创建由另一个 ERC721 支持的 ERC721 的包装器，具有存款和取款方法。与 ERC721Votes 结合使用很有用。