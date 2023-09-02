[Array-deletion.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Array-deletion.sol)

# 数组删除疏忽

**名称：** 数组删除疏忽：导致数据不一致

**描述：** 在 Solidity 中，从动态数组中不当地删除元素可能会导致数据不一致。 当尝试从数组中删除元素时，如果删除过程处理不当，数组可能仍保留存储空间并表现出意外行为。

**解决办法：**

选项1：复制最后一个元素并将其放置在要删除的位置。 选项2：将它们从右向左移动。

**参考：**

https://twitter.com/1nf0s3cpt/status/1677167550277509120

https://blog.solidityscan.com/improper-array-deletion-82672eed8e8d

https://github.com/sherlock-audit/2023-03-teller-judging/issues/88

**ArrayDeletionBug 漏洞合约样例:**

```jsx
contract ArrayDeletionBug {
    uint[] public myArray = [1, 2, 3, 4, 5];

    function deleteElement(uint index) external {
        require(index < myArray.length, "Invalid index");
        delete myArray[index];
    }

    function getLength() public view returns (uint) {
        return myArray.length;
    }
}
```

**FixedArrayDeletion 修复合约样例:（紫色字体)**

```jsx
contract FixedArrayDeletion {
    uint[] public myArray = [1, 2, 3, 4, 5];

    //解决措施1：通过复制最后一个元素并将其放置到要删除的位置。
    function deleteElement(uint index) external {
        require(index < myArray.length, "Invalid index");

        //将要删除的元素与最后一个元素交换
        myArray[index] = myArray[myArray.length - 1];

        //删除最后一个元素
        myArray.pop();
    }

    /*解决措施 2：将它们从右向左移动。
    function deleteElement(uint index) external {
        require(index < myArray.length, "Invalid index");
        
        for (uint i = _index; i < myArray.length - 1; i++) {
            myArray[i] = myArray[i + 1];
        }
        
        // 删除最后一个元素
        myArray.pop();
    }
```

***\*测试方法:\****

forge test --contracts src/test/**Array-deletion.sol**-vvvv

```jsx
// 测试函数以检查 ArrayDeletionBugContract 中的数组删除情况。
function testArrayDeletion() public {
    // 访问 ArrayDeletionBugContract 的“myArray”数组中索引 1 处的元素。
    ArrayDeletionBugContract.myArray(1);

    // 尝试从 'myArray' 数组中删除索引 1 处的元素，但这是不正确的
    // 注意：这里没有展示在 Solidity 中从数组中删除元素的正确方法
    ArrayDeletionBugContract.deleteElement(1);

    // 再次访问 ArrayDeletionBugContract 的 'myArray' 数组中索引 1 处的元素。
     // 但是，由于删除操作不正确，可能没有被正确删除。
    ArrayDeletionBugContract.myArray(1);

    // 获取 ArrayDeletionBugContract 中“myArray”数组的当前长度。
     // 这一步是检查数组的长度是否受到错误删除的影响。
    ArrayDeletionBugContract.getLength();
}
```

**红色框：**在Solidity中，从动态数组中不当删除元素可能会导致数据不一致。 当尝试从数组中删除元素时，如果删除过程处理不当，数组可能仍保留存储空间并表现出意外行为。

**紫色框：问题已修复。**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F8e3277c3-ea93-4173-87c0-253809371bed%2FUntitled.png?table=block&id=1e01fe04-3b56-4d8e-a4e0-ae880f66d86a&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)