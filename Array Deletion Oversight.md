# Array Deletion Oversight

**Name:** Array Deletion Oversight: leading to data inconsistency

**Description:** In Solidity where improper deletion of elements from dynamic arrays can result in data inconsistency. When attempting to delete elements from an array, if the deletion process is not handled correctly, the array may still retain storage space and exhibit unexpected behavior.

**Mitigation:**

Option1: By copying the last element and placing it in the position to be removed. Option2: By shifting them from right to left.

**REF:**

https://twitter.com/1nf0s3cpt/status/1677167550277509120

https://blog.solidityscan.com/improper-array-deletion-82672eed8e8d

https://github.com/sherlock-audit/2023-03-teller-judging/issues/88

**ArrayDeletionBug Contract:**

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

**FixedArrayDeletion Contract:（Purple words)**

```jsx
contract FixedArrayDeletion {
    uint[] public myArray = [1, 2, 3, 4, 5];

    //Mitigation 1: By copying the last element and placing it in the position to be removed.
    function deleteElement(uint index) external {
        require(index < myArray.length, "Invalid index");

        // Swap the element to be deleted with the last element
        myArray[index] = myArray[myArray.length - 1];

        // Delete the last element
        myArray.pop();
    }

    /*Mitigation 2: By shifting them from right to left.
    function deleteElement(uint index) external {
        require(index < myArray.length, "Invalid index");
        
        for (uint i = _index; i < myArray.length - 1; i++) {
            myArray[i] = myArray[i + 1];
        }
        
        // Delete the last element
        myArray.pop();
    }
```

***\*How to Test:\****

forge test --contracts src/test/**Array-deletion.sol**-vvvv

```jsx
// Test function to check array deletion in the ArrayDeletionBugContract.
function testArrayDeletion() public {
    // Access the element at index 1 in the 'myArray' array of the ArrayDeletionBugContract.
    ArrayDeletionBugContract.myArray(1);

    // Attempt to delete the element at index 1 from the 'myArray' array, but it is incorrect.
    // Note: The correct way to delete an element from an array in Solidity is not shown here.
    ArrayDeletionBugContract.deleteElement(1);

    // Access the element at index 1 in the 'myArray' array of the ArrayDeletionBugContract again.
    // However, it might not have been properly deleted due to the incorrect delete operation.
    ArrayDeletionBugContract.myArray(1);

    // Get the current length of the 'myArray' array in the ArrayDeletionBugContract.
    // This step is to check if the length of the array has been affected by the incorrect deletion.
    ArrayDeletionBugContract.getLength();
}
```

**Red box:**

In Solidity where improper deletion of elements from dynamic arrays can result in data inconsistency. When attempting to delete elements from an array, if the deletion process is not handled correctly, the array may still retain storage space and exhibit unexpected behavior.

**Purple box: issue fixed.**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F8e3277c3-ea93-4173-87c0-253809371bed%2FUntitled.png?table=block&id=1e01fe04-3b56-4d8e-a4e0-ae880f66d86a&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)