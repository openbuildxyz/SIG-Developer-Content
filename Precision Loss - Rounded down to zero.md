# Precision Loss - Rounded down to zero

**Name:** Precision Loss - rounding down to zero

**Description:** Support all the ERC20 tokens, as those tokens may have different decimal places. For example, USDT and USDC have 6 decimals. So, in the calculations, one must be careful.

**Mitigation:**

Avoid any situation that if the numerator is smaller than the denominator, the result will be zero. Rounding down related issues can be avoided in many ways: 1.Using libraries for rounding up/down as expected 2.Requiring result is not zero or denominator is <= numerator 3.Refactor operations for avoiding first dividing then multiplying, when first dividing then multiplying, precision lost is amplified

**REF:**

https://twitter.com/1nf0s3cpt/status/1675805135061286914

https://github.com/sherlock-audit/2023-02-surge-judging/issues/244

https://github.com/sherlock-audit/2023-02-surge-judging/issues/122

https://dacian.me/precision-loss-errors#heading-rounding-down-to-zero

**SimplePool Contract:**

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
        // Get the time passed since the last interest accrual
        uint _timeDelta = block.timestamp - lastAccrueInterestTime; //_timeDelta=1

        // If the time passed is 0, return 0 reward
        if (_timeDelta == 0) return 0;

        // Calculate the supplied value
        // uint _supplied = totalDebt + loanTokenBalance;
        //console.log(_supplied);
        // Calculate the reward
        _reward = (totalDebt * _timeDelta) / (365 days * 1e18);
        console.log("Current reward", _reward);

        // 31536000 is the number of seconds in a year
        // 365 days * 1e18 = 31_536_000_000_000_000_000_000_000
        //_totalDebt * _timeDelta / 31_536_000_000_000_000_000_000_000
        // 10_000_000_000 * 1 / 31_536_000_000_000_000_000_000_000 // -> 0
        _reward;
    }
}
```

***\*How to Test:\****

forge test --contracts src/test/**Precision-loss.sol** -vvvv

```jsx
// This function is declared as 'view' which means it will not modify the contract's state.
function testRounding_error() public view {
    // Calls the 'getCurrentReward' function from the 'SimplePoolContract' contract.
    SimplePoolContract.getCurrentReward();
}
```

**Red box: reward is rounding to zero.**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F23257271-d3d1-44fc-956b-079a7d73ba68%2FUntitled.png?table=block&id=578e2130-6152-405e-9884-5d4d89191a44&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)