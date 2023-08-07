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
        // createHash(AAA, BBB) -> AAABBB
        // createHash(AA, ABBB) -> AAABBB
        // Check if the hash already exists in the balances mapping
        require(balances[hash] == 0, "Hash collision detected");

        balances[hash] = msg.value;
    }
}
```

***\*测试方法:\****

仿真测试 --contracts src/test/**Hash-collisions.sol**-vvvv

```jsx
// Test function to check for hash collision bug in the HashCollisionBugContract.
function testHash_collisions() public {
    // Emit a log with the name and the result of the hash calculation for inputs "AAA" and "BBB".
    emit log_named_bytes32(
        "(AAA,BBB) Hash",
        HashCollisionBugContract.createHash("AAA", "BBB")
    );

    // Call the 'deposit' function of the HashCollisionBugContract with value 1 ether and inputs "AAA" and "BBB".
    HashCollisionBugContract.deposit{value: 1 ether}("AAA", "BBB");

    // Emit a log with the name and the result of the hash calculation for inputs "AA" and "ABBB".
    emit log_named_bytes32(
        "(AA,ABBB) Hash",
        HashCollisionBugContract.createHash("AA", "ABBB")
    );

    // Expect a revert with the message "Hash collision detected" for the following transaction.
    // Call the 'deposit' function of the HashCollisionBugContract with value 1 ether and inputs "AA" and "ABBB".
    // This transaction is expected to revert because it might trigger a hash collision bug.
    vm.expectRevert("Hash collision detected");
    HashCollisionBugContract.deposit{value: 1 ether}("AA", "ABBB"); //Hash collision detected
}
```

**红色框：相同的哈希值。**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fe334209d-2420-4a6d-a363-22cbb2d81756%2FUntitled.png?table=block&id=6e6f8553-0929-4f95-ad76-51f2f50ef00f&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)