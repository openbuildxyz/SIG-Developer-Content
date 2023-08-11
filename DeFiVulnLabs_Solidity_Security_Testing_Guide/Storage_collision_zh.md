# 存储碰撞  
[Storage-collision.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Storage-collision.sol)  
**名称：** 存储碰撞漏洞

**描述：**  
这个漏洞是由于Proxy合约和Logic合约都使用相同的存储槽（槽0）来存储重要变量，  
即Proxy合约中的implementation变量地址和Logic合约中的GuestAddress变量地址。  
由于Proxy合约使用delegatecall方法与Logic合约进行交互，  
他们共享相同的存储空间。如果调用foo函数，
它会覆盖Proxy合约中的implementation变量地址，从而导致意外行为。  

**缓解措施：**   
缓解此问题的一种方法是设计proxy和logic合约的存储布局保持一致。  

**参考：**  
https://blog.openzeppelin.com/proxy-patterns  

**Proxy合约：**  
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
**如何测试：**  
forge test --contracts src/test/Storage-collision.sol-vvvv  
```
// 用于测试存储碰撞漏洞的函数。
function testStorageCollision() public {
    // 创建LogicContract合约的实例。
    LogicContract = new Logic();

    // 创建Proxy的新实例，并将LogicContract的地址传递给其构造函数。
    ProxyContract = new Proxy(address(LogicContract));

    // 记录当前实现合约（LogicContract）的地址。
    console.log(
        "Current implementation contract address:",
        ProxyContract.implementation()
    );

   
    //调用 ProxyContract 的 testcollision() 函数。这个函数可能会改变用于存储 LogicContract 地址的存储槽。
    ProxyContract.testcollision();

    // 记录被覆写的实现合约地址。
    console.log(
        "Overwritten slot0 implementation contract address:",
        ProxyContract.implementation()
    );

    // 记录“攻击完成”。
    console.log("Exploit completed");
}
```
**红框：** 覆盖插槽0。  
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fc2707314-2098-477d-a55e-b5beb4301636%2FUntitled.png?table=block&id=32bfcdcc-533a-44a6-bc11-2f873408fffa&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)