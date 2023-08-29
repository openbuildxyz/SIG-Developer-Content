[Precision-loss.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Precision-loss.sol)

# 精度损失 - 向下舍入为零

**名称：** 精度损失 - 向下舍入为零

**描述：** 支持所有 ERC20 代币，因为这些代币可能有不同的小数位。 例如，USDT和USDC有6位小数。 所以，在计算的时候，一定要小心。

**解决办法：**

避免分子小于分母，结果为零的情况。 与下舍入相关的问题可以通过多种方式避免： 1.使用库按预期向上/向下舍入 2.要求结果不为零或分母 <= 分子 3.重构操作以避免先除后乘，先除后乘 乘法，精度损失被放大

**参考：**

https://twitter.com/1nf0s3cpt/status/1675805135061286914

https://github.com/sherlock-audit/2023-02-surge-judging/issues/244

https://github.com/sherlock-audit/2023-02-surge-judging/issues/122

https://dacian.me/precision-loss-errors#heading-rounding-down-to-zero

**SimplePool 合约样例:**

```jsx
contract SimplePool {
    uint public totalDebt;
    uint public lastAccrueInterestTime;
    uint public loanTokenBalance;

    constructor() {
        totalDebt = 10000e6; //debt token is USDC and has 6 digit decimals.
        lastAccrueInterestTime = block.timestamp - 1;
        loanTokenBalance = 500e18;
    }

    function getCurrentReward() public view returns (uint _reward) {
        // 获取自上次计息以来经过的时间
        uint _timeDelta = block.timestamp - lastAccrueInterestTime; //_timeDelta=1

        // 如果经过的时间为0，则返回0奖励
        if (_timeDelta == 0) return 0;

        // 计算提供的值
         // uint _supplied = 总债务 + LoanTokenBalance;
         //console.log(_supplied);
         // 计算奖励
        _reward = (totalDebt * _timeDelta) / (365 days * 1e18);
        console.log("Current reward", _reward);

        // 31536000 是一年的秒数
        // 365 天 * 1e18 = 31_536_000_000_000_000_000_000_000
        //_总债务 * _timeDelta / 31_536_000_000_000_000_000_000_000
        // 10_000_000_000 * 1 / 31_536_000_000_000_000_000_000_000 // -> 0
        _reward;
    }
}
```

***\*测试方法:\****

forge test--contracts src/test/**Precision-loss.sol** -vvvv

```jsx
// 该函数被声明为“view”，这意味着它不会修改合约的状态。
function testRounding_error() public view {
    // 从“SimplePoolContract”合约调用“getCurrentReward”函数。
    SimplePoolContract.getCurrentReward();
}
```

**红色框：奖励四舍五入为零。**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F23257271-d3d1-44fc-956b-079a7d73ba68%2FUntitled.png?table=block&id=578e2130-6152-405e-9884-5d4d89191a44&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)