# 自助转账支票缺失导致的资金丢失

**名称：** 自助转账支票缺失导致资金丢失

**描述：** 代码中的漏洞源于缺乏防止自助转账的检查。 这种疏忽导致转账功能错误地将资金转移到同一地址。 由于代码无法从发送者的余额中扣除转账金额，因此资金会丢失。 该漏洞破坏了合约内资金转移的正确性，并对用户余额的完整性构成风险。

**解决办法：**

添加条件以防止相同地址之间的传输

**参考：**

https://twitter.com/1nf0s3cpt/status/1679373800327241728

https://github.com/code-423n4/2022-10-traderjoe-findings/issues/299

https://www.immunebytes.com/blog/bzxs-security-focused-relaunch-followed-by-a-hack-how/

**样例合约:**

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

**漏洞解决合约:（紫色字体）**

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

***\*测试方法:\****

**仿真测试 --contracts src/test/self-transfer.sol -vvvv**

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

**红框：** 代码中的漏洞源于没有进行防止自助转账的检查。 **紫色框：**问题已修复。

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fc5af0213-401d-43e9-aa06-fc457ac0dc40%2FUntitled.png?table=block&id=08376c60-c818-47ca-9da1-fcfafdb5f707&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)