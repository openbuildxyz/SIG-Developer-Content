# Data location - storage vs memory
[DataLocation.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/DataLocation.sol)  
**Name:** Data Location Confusion Vulnerability

**Description:**   
Misuse of storage and memory references of the user in the updaterewardDebt function.

The function updaterewardDebt is updating the rewardDebt value of a UserInfo struct
that is stored in memory. The issue is that this won't persist between function calls.
As soon as the function finishes executing, the memory is cleared and the changes are lost.

**Mitigation:**  
Ensure the correct usage of memory and storage in the function parameters. Make all the locations explicit.

**REF:**

https://mudit.blog/cover-protocol-hack-analysis-tokens-minted-exploit/

https://www.educative.io/answers/storage-vs-memory-in-solidity

**Array Contract: (Purple Words is the fixed function)**  
```
contract Array is Test {
    mapping(address => UserInfo) public userInfo; // storage

    struct UserInfo {
        uint256 amount; // How many tokens got staked by user.
        uint256 rewardDebt; // Reward debt. See Explanation below.
    }

    function updaterewardDebt(uint amount) public {
        UserInfo memory user = userInfo[msg.sender]; // memory, vulnerable point
        user.rewardDebt = amount;
    }

    function fixedupdaterewardDebt(uint amount) public {
        UserInfo storage user = userInfo[msg.sender]; // storage
        user.rewardDebt = amount;
    }
}
```
**How to Test:**

forge test --contracts src/test/**DataLocation.sol**-vvvv  
```
// A function to demonstrate the difference between memory and storage data locations in Solidity.
function testDataLocation() public {
    // Simulate dealing 1 ether to Alice and Bob.
    address alice = vm.addr(1);
    address bob = vm.addr(2);
    vm.deal(address(alice), 1 ether);
    vm.deal(address(bob), 1 ether);

    // Create a new instance of the Array contract.
    ArrayContract = new Array();

    // Update the rewardDebt storage variable in the Array contract to 100.
    ArrayContract.updaterewardDebt(100); 

    // Retrieve the userInfo struct for the contract's address and print the rewardDebt variable.
    // Note that the rewardDebt should still be the initial value, as updaterewardDebt operates on a memory variable, not the storage one.
    (uint amount, uint rewardDebt) = ArrayContract.userInfo(address(this));
    console.log("Non-updated rewardDebt", rewardDebt);

    // Print a message.
    console.log("Update rewardDebt with storage");

    // Now use the fixedupdaterewardDebt function, which correctly updates the storage variable.
    ArrayContract.fixedupdaterewardDebt(100);

    // Retrieve the userInfo struct again, and print the rewardDebt variable.
    // This time the rewardDebt should be updated to 100.
    (uint newamount, uint newrewardDebt) = ArrayContract.userInfo(
        address(this)
    );
    console.log("Updated rewardDebt", newrewardDebt);
}
```
**Red box: incorrect updated rewardDebt.**

**Purple box: i**ssue fixed.
![Alt text](image-20.png)