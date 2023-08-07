# Struct Deletion Oversight

**Name:** Struct Deletion Oversight

**Description:** Incomplete struct deletion leaves residual data. If you delete a struct containing a mapping, the mapping won't be deleted.

The bug arises because Solidity's delete keyword does not reset the storage to its initial state but rather performs a partial reset. When delete myStructs[structId] is called, it only resets the id at mappingId to its default value 0, but the other flags in the mapping remain unchanged. Therefore, if the struct is deleted without deleting the mapping inside, the remaining flags will persist in storage.

**Mitigation:**

To fix this bug, you should delete the mapping inside the struct before deleting the struct itself.

**REF:**

https://twitter.com/1nf0s3cpt/status/1676836264245592065

https://docs.soliditylang.org/en/develop/types.html

**StructDeletionBug Contract:**

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

**FixedStructDeletion Contract:**

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
        // Check if all flags are deleted, then delete the mapping
        for (uint256 i = 0; i < 15; i++) {
            delete myStruct.flags[i];
        }
        delete myStructs[structId];
    }
}
```

***\*How to Test:\****

forge test --contracts src/test/**Struct-deletion.sol** -vvvv

```jsx
// Test function to check struct deletion in the StructDeletionBugContract.
function testStructDeletion() public {
    // Add a struct with ID 10 and values 10 and 10 to the StructDeletionBugContract.
    StructDeletionBugContract.addStruct(10, 10);

    // Get the struct with ID 10 and values 10 and 10 from the StructDeletionBugContract.
    StructDeletionBugContract.getStruct(10, 10);

    // Delete the struct with ID 10 from the StructDeletionBugContract.
    StructDeletionBugContract.deleteStruct(10);

    // Attempt to get the struct with ID 10 and values 10 and 10 from the StructDeletionBugContract after deletion.
    StructDeletionBugContract.getStruct(10, 10);
}
```

**Red box: the mapping inside the struct didn't delete before deleting the struct itself. Purple box: issue fixed.**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fb4ebe1c3-5b6a-4651-91f1-0d4c1ea46303%2FUntitled.png?table=block&id=627a5d17-734e-4399-8222-be7578529197&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)