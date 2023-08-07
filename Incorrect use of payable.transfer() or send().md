# Incorrect use of payable.transfer() or send()

**Name:** Incorrect use of payable.transfer()

**Description:** After the implementation of EIP 1884 in the Istanbul hard fork, the gas cost of the SLOAD operation was increased, resulting in the breaking of some existing smart contracts.

When transferring ETH to recipients, if Solidity's transfer() or send() method is used, certain shortcomings arise, particularly when the recipient is a smart contract. These shortcomings can make it impossible to successfully transfer ETH to the smart contract recipient.

Specifically, the transfer will inevitably fail when the smart contract: 1.does not implement a payable fallback function, or 2.implements a payable fallback function which would incur more than 2300 gas units, or 3.implements a payable fallback function incurring less than 2300 gas units but is called through a proxy that raises the call’s gas usage above 2300.

**Mitigation:**

Using call with its returned boolean checked in combination with re-entrancy guard is highly recommended.

**REF:**

https://twitter.com/1nf0s3cpt/status/1678958093273829376

https://consensys.net/diligence/blog/2019/09/stop-using-soliditys-transfer-now/

https://github.com/code-423n4/2022-12-escher-findings/issues/99

**SimpleBank Contract:**

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
        // the issue is here
        payable(msg.sender).transfer(amount);
    }
}
```

**FixedSimpleBank Contract: (Purple words)**

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

***\*How to Test:\****

forge test --contracts src/test/**payable-transfer.sol**-vvvv

```jsx
// Test function to check a transfer failure in the SimpleBankContract.
function testTransferFail() public {
    // Deposit 1 ether into the SimpleBankContract.
    SimpleBankContract.deposit{value: 1 ether}();

    // Assert that the balance in the SimpleBankContract is 1 ether.
    // If it fails, the test will terminate with an error.
    assertEq(SimpleBankContract.getBalance(), 1 ether);

    // Set an expectation that the next transaction should revert (fail).
    vm.expectRevert();

    // Try to withdraw 1 ether from the SimpleBankContract.
    // This transaction is expected to fail and revert, as it attempts to withdraw more than the contract's balance.
    SimpleBankContract.withdraw(1 ether);
}
```

**Red box:**

After the implementation of EIP 1884 in the Istanbul hard fork, the gas cost of the SLOAD operation was increased, resulting in the breaking of some existing smart contracts. **Purple box:** issue fixed.

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fd87dfd5f-fcae-4673-9922-3bf112565db1%2FUntitled.png?table=block&id=6d42b103-1253-4fe4-8846-e80207524fac&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)