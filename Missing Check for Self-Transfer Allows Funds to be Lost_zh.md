[self-transfer.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/self-transfer.sol)

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
        // 不检查自传输
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
        //解决办法
        **require(_from != _to, "Cannot transfer funds to the same address.");**

        uint256 _fromBalance = _balances[_from];
        uint256 _toBalance = _balances[_to];

        unchecked {
            _balances[_from] = _fromBalance - _amount;
            _balances[_to] = _toBalance + _amount;
            /*
            另外的解决办法
            _balances[_id][_from] -= _amount;
            _balances[_id][_to] += _amount;
            */
        }
    }
}
```

***\*测试方法:\****

**forge test --contracts src/test/self-transfer.sol -vvvv**

```jsx
//测试函数以检查 VSimpleBankContract 内的自转账。
function testSelfTransfer() public {
    // 执行从合约到自身的第一次自转账 10000 个单位。
    VSimpleBankContract.transfer(address(this), address(this), 10000);

    // 执行从合约到自身的第二次自转账 10000 单位。
    VSimpleBankContract.transfer(address(this), address(this), 10000);

    // 两次转账后获取地址 `this` 的合约余额。
     // 假设该余额代表“Alice”的总余额。
    VSimpleBankContract.balanceOf(address(this));

    // 代码中提到“Alice”的余额增加了10000，
     // 然后减少 10000，“Alice”的总余额为 20000。
     // 但是，需要澄清的是，这只是一条注释，而不是实际的代码执行。
     // 如果这是真正的代码，那么取消注释就会导致计算。
}
```

**红框：** 代码中的漏洞源于没有进行防止自助转账的检查。 **紫色框：**问题已修复。

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fc5af0213-401d-43e9-aa06-fc457ac0dc40%2FUntitled.png?table=block&id=08376c60-c818-47ca-9da1-fcfafdb5f707&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)