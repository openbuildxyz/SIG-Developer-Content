# 不变量  
[Invariant.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Invariant.sol)    
**名称：** 不变性问题  

**描述：**  
assert 用于检查不变性。这些不变性是指我们的合约或变量永远不应该达到的状态。  
例如，如果我们减少一个值，它就不应该变得更大，只能变得更小。  

在给定的代码中，Invariant 合约包含一个 receiveMoney 函数，  
用于接收以太币并将发送者的余额增加相应的金额。这个余额以 uint64 的形式存储。  
无符号整数可以存储从 0 到 2^n - 1 的值，在这种情况下是 2^64 - 1，大约是 18.4467 以太币。  
  

如果发送者发送的以太币超过了 uint64 可以存储的最大值，就会发生溢出，值会回滚到 0 并从那里开始递增。  
因此，余额不会准确地反映合约接收到的以太币数量。  

**缓解措施：**  
为了避免这个问题，重要的是要确保用于存储值的类型大小适合需要存储的值。  
**参考：**  
https://ethereum-blockchain-developer.com/027-exceptions/04-invariants-with-assert/  

**Invariant合约：**  
```
contract Invariant {
    mapping(address => uint64) public balanceReceived;

    function receiveMoney() public payable {
        balanceReceived[msg.sender] += uint64(msg.value);
    }

    function withdrawMoney(address payable _to, uint64 _amount) public {
        require(
            _amount <= balanceReceived[msg.sender],
            "Not Enough Funds, aborting"
        );

        balanceReceived[msg.sender] -= _amount;
        _to.transfer(_amount);
    }
}
``` 
**如何测试：**
forge test --contracts src/test/Invariant.sol -vvvv  
```
// 演示不变性问题的函数
function testInvariant() public {
    // 创建Invariant合约的新实例
    InvariantContract = new Invariant();

    // 发送1个以太币到合约，并打印余额
    InvariantContract.receiveMoney{value: 1 ether}();
    console.log(
        "BalanceReceived:",
        InvariantContract.balanceReceived(address(this))
    );

    // 再次发送18个以太币到合约，并打印余额
    // 合约现在应该有19个以太币，但由于整数溢出问题，实际上显示的余额不正确。
    InvariantContract.receiveMoney{value: 18 ether}();
    console.log(
        "testInvariant, BalanceReceived:",
        InvariantContract.balanceReceived(address(this))
    );
}
```
**红色框：** 绕过了不变式并发生了溢出。  

![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F1aebb37e-6e01-4f55-ab64-b94a8c4f61a6%2FUntitled.png?table=block&id=8b09f6be-ea97-40b5-ad6e-98cb785bb4d9&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)
