# 数据位置 - 存储 VS 内存
[DataLocation.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/DataLocation.sol)  

**名称：** 数据位置混淆漏洞   

**描述：**  
在 updaterewardDebt 函数中错误地使用了用户的storage和memory。

updaterewardDebt 函数更新存储在memory中的 UserInfo 结构体的 rewardDebt 值。  
问题在于，这些更新不会在函数调用之间持久化。一旦函数执行完成，内存会被清除，更改将丢失。


**缓解措施：**  
确保函数参数中正确使用了内存和存储位置。明确指定所有位置。  

**参考：**  
https://mudit.blog/cover-protocol-hack-analysis-tokens-minted-exploit/  
https://www.educative.io/answers/storage-vs-memory-in-solidity  


数组合约：（紫色字为修复函数）
```solidity
contract Array is Test {
    mapping(address => UserInfo) public userInfo; // 存储

    struct UserInfo {
        uint256 amount; // 用户质押了多少代币
        uint256 rewardDebt; // 奖励债务，请参阅下面的说明
    }

    function updaterewardDebt(uint amount) public {
        UserInfo memory user = userInfo[msg.sender]; // 内存，漏洞点
        user.rewardDebt = amount;
    }

    function fixedupdaterewardDebt(uint amount) public {
        UserInfo storage user = userInfo[msg.sender]; // 存储
        user.rewardDebt = amount;
    }
}
```  
**如何测试：**  
forge test --contracts src/test/DataLocation.sol-vvvv  

```
// 演示Solidity中内存和存储数据位置之间差异的函数
function testDataLocation() public {
    // 模拟向Alice和Bob分配1个以太币
    address alice = vm.addr(1);
    address bob = vm.addr(2);
    vm.deal(address(alice), 1 ether);
    vm.deal(address(bob), 1 ether);

    // 创建Array合约的新实例
    ArrayContract = new Array();

    // 将Array合约中的rewardDebt变量更新为100
    ArrayContract.updaterewardDebt(100); 

    // 检索合约地址的userInfo结构并打印rewardDebt变量
    // 请注意，rewardDebt仍应为初始值，因为 updaterewardDebt操作的是内存变量，而不是存储变量
    (uint amount, uint rewardDebt) = ArrayContract.userInfo(address(this));
    console.log("Non-updated rewardDebt", rewardDebt);

    // 打印一条消息
    console.log("Update rewardDebt with storage");

    // 现在使用fixedupdaterewardDebt函数，它可以正确更新存储变量
    ArrayContract.fixedupdaterewardDebt(100);

    // 再次检索userInfo结构体，并打印rewardDebt变量
    // 这次rewardDebt应该更新为100
    (uint newamount, uint newrewardDebt) = ArrayContract.userInfo(
        address(this)
    );
    console.log("Updated rewardDebt", newrewardDebt);
}
```  
**红框：** 错误更新的rewardDebt  
**紫框：** 问题已修复
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F1cf5afb4-a281-43e4-abf5-31831f3b874f%2FUntitled.png?table=block&id=ebe353c4-5056-4d72-aff6-064bde3c21a8&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)