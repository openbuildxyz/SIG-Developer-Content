[payable-transfer.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/payable-transfer.sol)

# payable.transfer() 或 send() 的错误使用

**名称：** payable.transfer() 的错误使用

**描述：** 伊斯坦布尔硬分叉的EIP 1884实施后，SLOAD操作的gas成本增加，导致部分现有智能合约被破坏。

当将 ETH 转移给接收者时，如果使用 Solidity 的 transfer() 或 send() 方法，则会出现某些缺陷，特别是当接收者是智能合约时。 这些缺陷可能导致 ETH 无法成功转移给智能合约的接收者。

具体来说，当智能合约出现以下情况时，转账将不可避免地失败： 1. 未实现应付回退功能； 2. 实现应付回退功能，将产生超过 2300 个 Gas 单位；3. 实现应付回退功能，产生少于 2300 个 Gas 单位，但通过代理调用，将调用的 Gas 使用量提高到 2300 以上。

**解决办法：**

强烈建议将返回的布尔值检查与重复进入防护结合使用。

**参考：**

https://twitter.com/1nf0s3cpt/status/1678958093273829376

https://consensys.net/diligence/blog/2019/09/stop-using-soliditys-transfer-now/

https://github.com/code-423n4/2022-12-escher-findings/issues/99

**合约样例:**

```jsx
contract SimpleBank {
    mapping(address => uint) private balances;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function getBalance() public view returns (uint) {
        return balances[msg.sender];
    }

    function withdraw(uint amount) public {
        require(balances[msg.sender] >= amount);
        balances[msg.sender] -= amount;
        // 问题在这
        payable(msg.sender).transfer(amount);
    }
}
```

**已修复合约样例 (紫色字体)**

```jsx
contract FixedSimpleBank {
    mapping(address => uint) private balances;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function getBalance() public view returns (uint) {
        return balances[msg.sender];
    }

    function withdraw(uint amount) public {
        require(balances[msg.sender] >= amount);
        balances[msg.sender] -= amount;
        **(bool success, ) = payable(msg.sender).call{value: amount}("");
        require(success, " Transfer of ETH Failed");**
    }
}
```

***\*测试方法:\****

**forge test --contracts src/test/**payable-transfer.sol**-vvvv**

```jsx
//测试函数以检查 SimpleBankContract 中的转账失败。
function testTransferFail() public {
    //将 1 以太币存入 SimpleBankContract 中。
    SimpleBankContract.deposit{value: 1 ether}();

    // 判断 SimpleBankContract 中的余额为 1 ether。
     // 如果失败，测试将因错误而终止。
    assertEq(SimpleBankContract.getBalance(), 1 ether);

    // 设置下一个事务应该恢复（失败）的期望。
    vm.expectRevert();

    // 尝试从 SimpleBankContract 中提取 1 个以太币。
     // 该交易预计会失败并恢复，因为它尝试提取的金额超过了合约的余额。
    SimpleBankContract.withdraw(1 ether);
}
```

**红框：**伊斯坦布尔硬分叉实施EIP 1884后，SLOAD操作的gas成本增加，导致部分现有智能合约被破坏。 **紫色框：**问题已修复。

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fd87dfd5f-fcae-4673-9922-3bf112565db1%2FUntitled.png?table=block&id=6d42b103-1253-4fe4-8846-e80207524fac&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)