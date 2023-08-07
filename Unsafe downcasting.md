# Unsafe downcasting

**Name:** Unsafe downcasting

**Description:** Downcasting from a larger integer type to a smaller one without checks can lead to unexpected behavior if the value of the larger integer is outside the range of the smaller one.

**Mitigation:**

Make sure consistent uint256, or use openzepplin safeCasting.

**REF:**

https://twitter.com/1nf0s3cpt/status/1673511868839886849

https://github.com/code-423n4/2022-12-escher-findings/issues/369

https://github.com/sherlock-audit/2022-10-union-finance-judging/issues/96

**SimpleBank Contract:**

```jsx
contract SimpleBank {
    mapping(address => uint) private balances;

    function deposit(uint256 amount) public {
        // Here's the unsafe downcast. If the `amount` is greater than type(uint8).max
        // (which is 255), then only the least significant 8 bits are stored in balance.
        // This could lead to unexpected results due to overflow.
        uint8 balance = uint8(amount);

        // store the balance
        balances[msg.sender] = balance;
    }

    function getBalance() public view returns (uint) {
        return balances[msg.sender];
    }
}
```

**FixedSimpleBank Contract:(**Purple Words)

```jsx
contract FixedSimpleBank {
    using SafeCast for uint256; // Use SafeCast for uint256

    mapping(address => uint) private balances;

    function deposit(uint256 _amount) public {
        // Use the `toUint8()` function from `SafeCast` to safely downcast `amount`.
        // If `amount` is greater than `type(uint8).max`, it will revert.
        // or keep the same uint256 with amount.
        uint8 amount = _amount.toUint8(); // or keep uint256

        // Store the balance
        balances[msg.sender] = amount;
    }

    function getBalance() public view returns (uint) {
        return balances[msg.sender];
    }
}
```

***\*How to Test:\****

forge test --contracts src/test/***\*unsafe-downcast.sol\**** -vvvv

```jsx
// Function to test the potential risks of an unsafe type downcast
    function testUnsafeDowncast() public {
        // Deposits 257 units of a token (presumably wei if it's a typical ETH token) in the SimpleBankContract
        // If the SimpleBankContract stores this value in a uint8 variable, it will overflow and only store 1 (257 mod 256)
        SimpleBankContract.deposit(257); //overflowed

        // Logs the balance of the SimpleBankContract
        // If the overflow occurred, it would show 1 despite the deposit of 257
        console.log(
            "balance of SimpleBankContract:",
            SimpleBankContract.getBalance()
        );

        // Asserts that the balance of the SimpleBankContract should be 1
        // If it's not, the function will throw an error
        // If an overflow occurred due to an unsafe downcast to uint8, the balance will indeed be 1
        assertEq(SimpleBankContract.getBalance(), 1);
    }
```

**Red box: Downcasting from a larger integer type to a smaller one without checks. Purple box: issue fixed.**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fc2ebf979-6ed5-4301-b08c-1a714a83e101%2FUntitled.png?table=block&id=492fedcd-35d6-4e85-9709-f64cacaeda3a&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)