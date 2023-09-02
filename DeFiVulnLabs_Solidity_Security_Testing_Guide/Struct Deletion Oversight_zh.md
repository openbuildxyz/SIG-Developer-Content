[Struct-deletion.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Struct-deletion.sol)

# 结构删除疏忽

**名称：** 结构删除疏忽

**描述：** 不完整的结构删除会留下残留数据，如果删除包含映射的结构，则不会删除该映射。

该错误的出现是因为 Solidity 的删除关键字不会将存储重置为其初始状态，而是执行部分重置。 当调用delete myStructs[structId]时，它仅将mappingId处的id重置为其默认值0，但映射中的其他标志保持不变。 因此，如果删除结构体而不删除内部映射，则剩余标志将保留在存储中。

**解决办法：**

要修复此错误，您应该在删除结构体本身之前删除结构体内部的映射。

**参考：**

https://twitter.com/1nf0s3cpt/status/1676836264245592065

https://docs.soliditylang.org/en/develop/types.html

**StructDeletionBug 漏洞合约样例:**

```jsx
contract StructDeletionBug {
    struct MyStruct {
        uint256 id;
        mapping(uint256 => bool) flags;
    }

    mapping(uint256 => MyStruct) public myStructs;

    function addStruct(uint256 structId, uint256 flagKeys) public {
        MyStruct storage newStruct = myStructs[structId];
        newStruct.id = structId;
        newStruct.flags[flagKeys] = true;
    }

    function getStruct(
        uint256 structId,
        uint256 flagKeys
    ) public view returns (uint256, bool) {
        MyStruct storage myStruct = myStructs[structId];
        bool keys = myStruct.flags[flagKeys];
        return (myStruct.id, keys);
    }

    function deleteStruct(uint256 structId) public {
        MyStruct storage myStruct = myStructs[structId];
        delete myStructs[structId];
    }
}
```

**FixedStructDeletion 修复合约样例:**

```jsx
contract FixedStructDeletion {
    struct MyStruct {
        uint256 id;
        mapping(uint256 => bool) flags;
    }

    mapping(uint256 => MyStruct) public myStructs;

    function addStruct(uint256 structId, uint256 flagKeys) public {
        MyStruct storage newStruct = myStructs[structId];
        newStruct.id = structId;
        newStruct.flags[flagKeys] = true;
    }

    function getStruct(
        uint256 structId,
        uint256 flagKeys
    ) public view returns (uint256, bool) {
        MyStruct storage myStruct = myStructs[structId];
        bool keys = myStruct.flags[flagKeys];
        return (myStruct.id, keys);
    }

    function deleteStruct(uint256 structId) public {
        MyStruct storage myStruct = myStructs[structId];
        // 检查是否所有标志都被删除，然后删除映射
        for (uint256 i = 0; i < 15; i++) {
            delete myStruct.flags[i];
        }
        delete myStructs[structId];
    }
}
```

***\*测试方法:\****

**forge test --contracts src/test/**Struct-deletion.sol** -vvvv

```jsx
// 测试函数以检查 StructDeletionBugContract 中的结构删除。
function testStructDeletion() public {
    //将 ID 为 10、值为 10 和 10 的结构添加到 StructDeletionBugContract。
    StructDeletionBugContract.addStruct(10, 10);

    // 从 StructDeletionBugContract 获取 ID 为 10 且值为 10 和 10 的结构。
    StructDeletionBugContract.getStruct(10, 10);

    // 从 StructDeletionBugContract 中删除 ID 为 10 的结构体。
    StructDeletionBugContract.deleteStruct(10);

    // 删除后尝试从 StructDeletionBugContract 中获取 ID 为 10、值为 10 和 10 的结构体。
    StructDeletionBugContract.getStruct(10, 10);
}
```

**红色框：在删除结构体本身之前，结构体内部的映射没有删除。 紫色框：问题已修复。**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fb4ebe1c3-5b6a-4651-91f1-0d4c1ea46303%2FUntitled.png?table=block&id=627a5d17-734e-4399-8222-be7578529197&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)