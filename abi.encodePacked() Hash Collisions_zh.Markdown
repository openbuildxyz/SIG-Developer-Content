[Hash-collisions.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Hash-collisions.sol)

# abi.encodePacked() 哈希冲突

**名称：** abi.encodePacked() 哈希冲突

**说明：** 在某些情况下，将 abi.encodePacked() 与多个可变长度参数一起使用可能会导致哈希冲突。

哈希函数被设计为每个输入都是唯一的，但由于哈希函数大小或可能输入的绝对数量的限制，冲突仍然可能发生。 这是提到的已知问题：https://docs.soliditylang.org/en/v0.8.17/abi-spec.html?highlight=collisions#non-standard-packed-mode 这是提到的已知问题：https:// /docs.soliditylang.org/en/v0.8.17/abi-spec.html?highlight=collisions#non-standard-packed-mode

存款功能允许用户根据两个字符串输入将以太币存入合约：_string1 和 _string2。 该合约使用 keccak256 函数通过连接这两个字符串来生成唯一的哈希值。

如果 _string1 和 _string2 的两个不同组合产生相同的哈希值，则会发生哈希冲突。 该代码无法正确处理这种情况，并允许第二个存款人覆盖之前的存款。

**缓解措施：** 使用 abi.encode() 而不是 abi.encodePacked()

**参考：**

https://twitter.com/1nf0s3cpt/status/1676476475191750656

https://docs.soliditylang.org/en/v0.8.17/abi-spec.html?highlight=collisions#non-standard-packed-mode

https://swcregistry.io/docs/SWC-133

https://github.com/sherlock-audit/2022-10-nftport-judging/issues/118

**HashCollisionBug 漏洞合约:**

```jsx
contract HashCollisionBug {
    mapping(bytes32 => uint256) public balances;

    function createHash(
        string memory _string1,
        string memory _string2
    ) public pure returns (bytes32) {
        return keccak256(abi.encodePacked(_string1, _string2));
    }

    function deposit(
        string memory _string1,
        string memory _string2
    ) external payable {
        require(msg.value > 0, "Deposit amount must be greater than zero");

        bytes32 hash = createHash(_string1, _string2);
        // 创建Hash(AAA, BBB) -> AAABBB
        // 创建Hash(AA, ABBB) -> AAABBB
        // 检查余额映射中是否已存在哈希值
        require(balances[hash] == 0, "Hash collision detected");

        balances[hash] = msg.value;
    }
}
```

***\*测试方法:\****

forge test --contracts src/test/**Hash-collisions.sol**-vvvv

```jsx
// 测试函数以检查 HashCollisionBugContract 中的哈希冲突错误。
function testHash_collisions() public {
    // 发出一个日志，其中包含输入“AAA”和“BBB”的名称和哈希计算结果。
    emit log_named_bytes32(
        "(AAA,BBB) Hash",
        HashCollisionBugContract.createHash("AAA", "BBB")
    );

    // 调用 HashCollisionBugContract 的“deposit”函数，值为 1 ether 并输入“AAA”和“BBB”。
    HashCollisionBugContract.deposit{value: 1 ether}("AAA", "BBB");

    // 调用 HashCollisionBugContract 的“deposit”函数，值为 1 ether 并输入“AAA”和“BBB”。
    emit log_named_bytes32(
        "(AA,ABBB) Hash",
        HashCollisionBugContract.createHash("AA", "ABBB")
    );

    // 预计以下事务将返回“检测到哈希冲突”消息。
     // 使用值为 1 ether 并输入“AA”和“ABBB”来调用 HashCollisionBugContract 的“deposit”函数。
     // 该事务预计会恢复，因为它可能会触发哈希冲突错误。
    vm.expectRevert("Hash collision detected");
    HashCollisionBugContract.deposit{value: 1 ether}("AA", "ABBB"); //Hash collision detected
}
```

**红色框：相同的哈希值。**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fe334209d-2420-4a6d-a363-22cbb2d81756%2FUntitled.png?table=block&id=6e6f8553-0929-4f95-ad76-51f2f50ef00f&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)