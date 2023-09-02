[gas-price.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/gas-price.sol)

# txGasPrice 值操作

**名称：** txGasPrice 值操纵

**描述：** 操纵 txGasPrice 值，这可能会导致意外后果和潜在的财务损失。

在calculateTotalFee函数中，总费用是通过将gasUsed + GAS_OVERHEAD_NATIVE与txGasPrice相乘来计算的。 问题是 txGasPrice 值可能被攻击者操纵，可能导致费用计算夸大。

**解决办法：**

为了解决此漏洞，建议实施保护措施，例如使用 Gas 预言机从可信来源获取平均 Gas 价格。

**参考：**

https://twitter.com/1nf0s3cpt/status/1678268482641870849

https://github.com/solodit/solodit_content/blob/main/reports/ZachObront/2023-03-21-Alligator.md

https://github.com/solodit/solodit_content/blob/main/reports/Trust Security/2023-05-15-Brahma.md

https://blog.pessimistic.io/ethereum-alarm-clock-exploit-final-thoughts-21334987c331

**GasReimbursement 合约样例:**

```jsx
contract GasReimbursement {
    uint public gasUsed = 100000; // 假设使用的gas为100,000
    uint public GAS_OVERHEAD_NATIVE = 500; // 假设原生代币 Gas 开销为 500

    // uint public txGasPrice = 20000000000;  // 假设交易 Gas 价格为 20 gwei

    function calculateTotalFee() public view returns (uint) {
        uint256 totalFee = (gasUsed + GAS_OVERHEAD_NATIVE) * tx.gasprice;
        return totalFee;
    }

    function executeTransfer(address recipient) public {
        uint256 totalFee = calculateTotalFee();
        _nativeTransferExec(recipient, totalFee);
    }

    function _nativeTransferExec(address recipient, uint256 amount) internal {
        payable(recipient).transfer(amount);
    }
}
```

***\*测试方法:\****

**forge test --contracts src/test/**gas-price.sol**-vvvv**

```jsx
// 测试函数以检查 GasReimbursementContract 中的 Gas 退款。
function testGasRefund() public {
    //在执行转账之前获取该合约的余额。
    uint balanceBefore = address(this).balance;

    // 调用 GasReimbursementContract 的 'executeTransfer' 函数，这应该会触发 Gas 退款。
    GasReimbursementContract.executeTransfer(address(this));

    // 计算转账后该合约的余额，并从中减去gas费用（gas价格）。
    uint balanceAfter = address(this).balance - tx.gasprice; // --gas-price 200000000000000

    // 计算并记录合约从gas退款中获得的利润。
    console.log("Profit", balanceAfter - balanceBefore);
}
```

**红框：gas退款错误**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F1a5cb283-c5d4-4bd2-b802-5ebb285537d3%2FUntitled.png?table=block&id=6fd4f616-4158-4f5a-9ae6-32cc3659b1e6&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)