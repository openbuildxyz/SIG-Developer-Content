# Storage collision
[Storage-collision.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Storage-collision.sol)  
**Name:** Storage Collision Vulnerability

**Description:**  
The vulnerability is that both the Proxy and Logic contracts use the same storage slot (slot 0) to store important variables,
namely the implementation address in the Proxy contract and the GuestAddress in the Logic contract.
Since the Proxy contract is using the delegatecall method to interact with the Logic contract,
they share the same storage. If the foo function is called,
it overwrites the implementation address in the Proxy contract, which results in an unexpected behavior.

**Mitigation:**  
One approach to mitigating this issue is to design the storage layout of the proxy and logic contracts to be consistent with each other.

**REF:**

https://blog.openzeppelin.com/proxy-patterns

**Proxy** **Contract:**
```
contract Proxy {
    address public implementation; //slot0

    constructor(address _implementation) {
        implementation = _implementation;
    }

    function testcollision() public {
        bool success;
        (success, ) = implementation.delegatecall(
            abi.encodeWithSignature("foo(address)", address(this))
        );
    }
}

contract Logic {
    address public GuestAddress; //slot0

    constructor() {
        GuestAddress = address(0x0);
    }

    function foo(address _addr) public {
        GuestAddress = _addr;
    }
}
```
**How to Test:**

forge test --contracts src/test/**Storage-collision.sol**-vvvv  
```
// A function to test storage collision exploit.
function testStorageCollision() public {
    // A new instance of LogicContract is created.
    LogicContract = new Logic();

    // A new instance of ProxyContract is created, and the address of the LogicContract is passed to its constructor.
    ProxyContract = new Proxy(address(LogicContract));

    // Logs the current address of the implementation contract (LogicContract).
    console.log(
        "Current implementation contract address:",
        ProxyContract.implementation()
    );

    // Calls the testcollision() function of the ProxyContract. This function presumably changes the storage slot that is also used to store the address of the LogicContract.
    ProxyContract.testcollision();

    // Logs the overwritten address of the implementation contract.
    console.log(
        "Overwritten slot0 implementation contract address:",
        ProxyContract.implementation()
    );

    // Logs "Exploit completed".
    console.log("Exploit completed");
}
```
Red box: overwritten slot0.
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fc2707314-2098-477d-a55e-b5beb4301636%2FUntitled.png?table=block&id=32bfcdcc-533a-44a6-bc11-2f873408fffa&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)