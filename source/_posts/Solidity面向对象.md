---
title: Solidity面向对象
date: 2023-05-17 10:01:02
tags:
- Solidity
- Smart Contract
---

## inheritance、virtual、override

1. 合约之间存在继承关系，使用 `is`
1. 如果父合约的方法想被子合约继承则需要使用关键字 `virtual`
1. 如果子合约想要覆盖父合约的方法则需使用关键字 `override`
1. 在子合约中如果想调用父合约的方法需要使用 `super`
1. 继承遵循最远继承，即后面继承的合约会覆盖前面父合约的方法
1. super会调用继承链条上每一个合约的相关函数，而不仅仅是最近的父合约。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.10;

/* Inheritance tree
   A
 /  \
B   C
 \ /
  D
*/

contract A {
	event Log(string message);
	
	function foo() public virtual {
		emit Log("A.foo called");
	}
	function bar() public virtual {
		emit Log("A.bar called");
	}
}

contract B is A {
	function foo() public virtual override {
		emit Log("B.foo called");
		A.foo();
	}
	function bar() public virtual override {
		emit Log("B.bar called");
		super.bar();
	}
}

contract C is A {
    function foo() public virtual override {
        emit Log("C.foo called");
        A.foo();
    }

    function bar() public virtual override {
        emit Log("C.bar called");
        super.bar();
    }
}

contract D is B, C {
    // Try:
    // - Call D.foo and check the transaction logs.
    //   Although D inherits A, B and C, it only called C and then A.


    function foo() public override(B, C) {
        super.foo();
    }

      // Try:
    // - Call D.bar and check the transaction logs
    //   D called C, then B, and finally A.
    //   Although super was called twice (by B and C) it only called A once.
    function bar() public override(B, C) {
        super.bar();
    }
}
```

1. D调用foo的时候，由于B，C中的foo都没有使用super，所以只是覆盖问题，根据最远继承，C 覆盖了B，所以执行顺序为：D -> C -> A；
2. D调用bar的时候，由于B，C中的bar使用了super，此时D的两个parent都需要执行一遍，因此为D-> C -> B -> A，整个过程中A合约只会被调用一次。具体原因是Solidity借鉴了Python的方式，强制一个由基类构成的DAG（有向无环图）使其保证一个特定的顺序。

## 继承的状态变量覆盖

1. 状态变量无法进行重写，只能继承父类状态变量
2. 但是可以通过在子合约的构造函数中进行覆盖，从而达到重写目的

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

contract A {
    string public name = "Contract A";

    function getName() public view returns (string memory) {
        return name;
    }
}

// Shadowing is disallowed in Solidity 0.6
// This will not compile
// contract B is A {
//     string public name = "Contract B";
// }

contract C is A {
    // This is the correct way to override inherited state variables.
    constructor() {
        name = "Contract C";
    }

    // C.getName returns "Contract C"
}
```

## 构造函数

在继承合约时如果父合约有构造函数则需要显示对父合约进行构造。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

contract X {
	string public name;
	constructor(string memory _name) {
		name = _name;
	}
}

contract Y {
	string public text;
	
	constructor(string memory _name) {
		text = _name;
	}
}


// 第一种初始化方式
contract B is X("Input to X"), Y("Input to Y") {
	
}

// 构造顺序取决于继承顺序，从左到右，而不是实例化顺序
// 第二种初始化方式
contract D is X, Y {
	    constructor() X("X was called") Y("Y was called") {}
}
```

## Abstract

抽象合约的作用是将函数定义和具体实现分离，从而实现解耦、可拓展性，其使用规则为：

1. 当合约中有未实现的函数时，则合约必须修饰为abstract；
2. 当合约继承的base合约中有构造函数，但是当前合约并没有对其进行传参时，则必须修饰为abstract；
3. abstract合约中未实现的函数必须在子合约中实现，即所有在abstract中定义的函数都必须有实现；
4. abstract合约不能单独部署，必须被继承后才能部署；

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.0 <0.9.0;

abstract contract Animal {
    string public species;
    constructor(string memory _base) {
        species = _base;
    }
}

abstract contract Feline {
    uint public num;
    function utterance() public pure virtual returns (bytes32);

    function base(uint _num) public returns(uint, string memory) {
        num = _num;
        return (num, "hello world!");
    }
}

// 由于Animal中的构造函数没有进行初始化，所以必须修饰为abstract
abstract contract Cat1 is Feline, Animal {
    function utterance() public pure override returns (bytes32) { 
      return "miaow"; 
    }
}

contract Cat2 is Feline, Animal("Animal") {
    function utterance() public pure override returns (bytes32) { 
      return "miaow"; 
    }
}
```

## Interface

可以使用Interface完成多个合约之间进行交互，interface有如下特性：

1. 接口中定义的function不能存在具体实现；
2. 接口可以继承；
3. 所有的function必须定义为external；
4. 接口中不能存在constructor函数；
5. **接口中不能定义状态变量。**
6. [abstract和interface的区别](https://medium.com/upstate-interactive/solidity-how-to-know-when-to-use-abstract-contracts-vs-interfaces-874cab860c56)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

contract Counter {
    uint public count;

    function increment() external {
        count += 1;
    }
}

interface IBase {
    function count() external view returns (uint);
}

interface ICounter is IBase {
    function increment() external;
}

contract MyContract {
    function incrementCounter(address _counter) external {
        ICounter(_counter).increment();
    }

    function getCount(address _counter) external view returns (uint) {
        return ICounter(_counter).count();
    }
}
uniswap demo:

// Uniswap example
interface UniswapV2Factory {
    function getPair(address tokenA, address tokenB)
        external
        view
        returns (address pair);
}

interface UniswapV2Pair {
    function getReserves()
        external
        view
        returns (
            uint112 reserve0,
            uint112 reserve1,
            uint32 blockTimestampLast
        );
}

contract UniswapExample {
    address private factory = 0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f;
    address private dai = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    address private weth = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;

    function getTokenReserves() external view returns (uint, uint) {
        address pair = UniswapV2Factory(factory).getPair(dai, weth);
        (uint reserve0, uint reserve1, ) = UniswapV2Pair(pair).getReserves();
        return (reserve0, reserve1);
    }
}
```

