# Missing Check for Self-Transfer Allows Funds to be Lost

**Name:** Missing Check for Self-Transfer Allows Funds to be Lost

**Description:** The vulnerability in the code stems from the absence of a check to prevent self-transfers. This oversight allows the transfer function to erroneously transfer funds to the same address. Consequently, funds are lost as the code fails to deduct the transferred amount from the sender's balance. This vulnerability undermines the correctness of fund transfers within the contract and poses a risk to the integrity of user balances.

**Mitigation:**

Add condition to prevent transfer between same addresses

**REF:**

https://twitter.com/1nf0s3cpt/status/1679373800327241728

https://github.com/code-423n4/2022-10-traderjoe-findings/issues/299

https://www.immunebytes.com/blog/bzxs-security-focused-relaunch-followed-by-a-hack-how/

**SimpleBank Contract:**

```jsx
contract SimpleBank {
    mapping(address => uint256) private _balances;

    function balanceOf(address _account) public view virtual returns (uint256) {
        return _balances[_account];
    }

    function transfer(address _from, address _to, uint256 _amount) public {
        // not check self-transfer
        uint256 _fromBalance = _balances[_from];
        uint256 _toBalance = _balances[_to];

        unchecked {
            _balances[_from] = _fromBalance - _amount;
            _balances[_to] = _toBalance + _amount;
        }
    }
}
```

**FixedSimpleBank Contract:(**Purple Words)

```jsx
contract FixedSimpleBank {
    mapping(address => uint256) private _balances;

    function balanceOf(address _account) public view virtual returns (uint256) {
        return _balances[_account];
    }

    function transfer(address _from, address _to, uint256 _amount) public {
        //Mitigation
        **require(_from != _to, "Cannot transfer funds to the same address.");**

        uint256 _fromBalance = _balances[_from];
        uint256 _toBalance = _balances[_to];

        unchecked {
            _balances[_from] = _fromBalance - _amount;
            _balances[_to] = _toBalance + _amount;
            /*
            Another mitigation
            _balances[_id][_from] -= _amount;
            _balances[_id][_to] += _amount;
            */
        }
    }
}
```

***\*How to Test:\****

forge test --contracts src/test/self-transfer.sol -vvvv

```jsx
// Test function to check self-transfer within VSimpleBankContract.
function testSelfTransfer() public {
    // Perform the first self-transfer of 10000 units from the contract to itself.
    VSimpleBankContract.transfer(address(this), address(this), 10000);

    // Perform the second self-transfer of 10000 units from the contract to itself.
    VSimpleBankContract.transfer(address(this), address(this), 10000);

    // Get the balance of the contract at address `this` after the two transfers.
    // It's assumed that this balance represents the total balance of "Alice."
    VSimpleBankContract.balanceOf(address(this));

    // The code mentions that the balance of "Alice" is increased by 10000,
    // then decreased by 10000, resulting in a total balance of 20000 for "Alice."
    // However, it's important to clarify that this is just a comment and not actual code execution.
    // If this is meant to be real code, then uncommenting it would result in the calculations.
}
```

**Red box:** The vulnerability in the code stems from the absence of a check to prevent self-transfers. **Purple box:** issue fixed.

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fc5af0213-401d-43e9-aa06-fc457ac0dc40%2FUntitled.png?table=block&id=08376c60-c818-47ca-9da1-fcfafdb5f707&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)