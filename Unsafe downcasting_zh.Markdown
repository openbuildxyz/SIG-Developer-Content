[unsafe-downcast.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/unsafe-downcast.sol)

# 不安全的向下转型

**名称：** 不安全的向下转型

**说明：** 如果较大整数的值超出较小整数的范围，则在不进行检查的情况下从较大整数类型向下转换为较小整数类型可能会导致意外行为。

**减轻：**

确保uint256一致，或者使用openzepplin safeCasting。

**参考：**

https://twitter.com/1nf0s3cpt/status/1673511868839886849

https://github.com/code-423n4/2022-12-escher-findings/issues/369

https://github.com/sherlock-audit/2022-10-union-finance-judging/issues/96

**SimpleBank 合约样例:**

```jsx
contract SimpleBank {
    mapping(address => uint) private balances;

    function deposit(uint256 amount) public {
        // 这是不安全的向下转型。 如果 `amount` 大于 type(uint8).max
         //（即255），那么只有最低有效的8位被存储在平衡中。
         // 这可能会因溢出而导致意外结果。
        balances[msg.sender] = balance;
    }

    function getBalance() public view returns (uint) {
        return balances[msg.sender];
    }
}
```

**FixedSimpleBank 修复合约样例 (紫色字体)**

```jsx
contract FixedSimpleBank {
    using SafeCast for uint256; // 对 uint256 使用 SafeCast

    mapping(address => uint) private balances;

    function deposit(uint256 _amount) public {
        //使用“SafeCast”中的“toUint8()”函数安全地向下转换“amount”。
         // 如果 `amount` 大于 `type(uint8).max`，它将恢复。
         // 或保留与 amount 相同的 uint256。
        uint8 amount = _amount.toUint8(); // 或保留 uint256

        //存储余额
        balances[msg.sender] = amount;
    }

    function getBalance() public view returns (uint) {
        return balances[msg.sender];
    }
}
```

***\*测试方法:\****

**forge test--contracts src/test/***\*unsafe-downcast.sol\**** -vvvv

```jsx
// 测试不安全类型向下转型潜在风险的函数
    function testUnsafeDowncast() public {
        // 在 SimpleBankContract 中存入 257 个单位的代币（如果是典型的 ETH 代币，大概是 wei）
         // 如果 SimpleBankContract 将此值存储在 uint8 变量中，它将溢出并仅存储 1 (257 mod 256)
        SimpleBankContract.deposit(257); //overflowed

        // 记录 SimpleBankContract 的余额
         // 如果发生溢出，尽管存了257，但仍显示1
        console.log(
            "balance of SimpleBankContract:",
            SimpleBankContract.getBalance()
        );

        // 判断 SimpleBankContract 的余额应该为 1
         // 如果不是，函数会抛出错误
         // 如果由于不安全向下转型为 uint8 而发生溢出，则余额确实为 1
        assertEq(SimpleBankContract.getBalance(), 1);
    }
```

**红色框：从较大的整数类型向下转换为较小的整数类型未进行检查。 紫色框：问题已修复。**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fc2ebf979-6ed5-4301-b08c-1a714a83e101%2FUntitled.png?table=block&id=492fedcd-35d6-4e85-9709-f64cacaeda3a&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)