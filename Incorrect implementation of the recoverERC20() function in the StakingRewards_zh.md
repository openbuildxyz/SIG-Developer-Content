[recoverERC20.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/recoverERC20.sol)

# StakeRewards 中的recoverERC20()函数的错误实现

**名称：** StakeRewards 中的recoverERC20()函数的错误实现

**描述：** StakeRewards.sol 中的recoverERC20() 函数可以作为所有者检索rewardsToken 的后门，当前没有针对rewardsToken进行相应的检查。 这创造了一种管理特权，所有者可以轻易获取奖励代币，并有可能将其用作压榨储户的手段。 如果您分叉了易受攻击的代码，这与分叉问题类似。

**解决办法：**

禁止在recoverErc20函数中恢复rewardToken

**参考：**

https://twitter.com/1nf0s3cpt/status/1680806251482189824

https://github.com/code-423n4/2022-02-concur-findings/issues/210

https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/49

https://github.com/code-423n4/2022-10-paladin-findings/issues/40

https://blog.openzeppelin.com/across-token-and-token-distributor-audit#anyone-can-prevent-stakers-from-getting-their-rewards

**VulnStakingRewards 合约:**

```jsx
contract VulnStakingRewards {
    using SafeERC20 for IERC20;

    IERC20 public rewardsToken;
    address public owner;

    event Recovered(address token, uint256 amount);

    constructor(address _rewardsToken) {
        rewardsToken = IERC20(_rewardsToken);
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function");
        _;
    }

    function recoverERC20(
        address tokenAddress,
        uint256 tokenAmount
    ) public onlyOwner {
        IERC20(tokenAddress).safeTransfer(owner, tokenAmount);
        emit Recovered(tokenAddress, tokenAmount);
    }
}
```

**问题修复合约:(紫色字体)**

```jsx
contract FixedtakingRewards {
    using SafeERC20 for IERC20;

    IERC20 public rewardsToken;
    address public owner;

    event Recovered(address token, uint256 amount);

    constructor(address _rewardsToken) {
        rewardsToken = IERC20(_rewardsToken);
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function");
        _;
    }

    function recoverERC20(
        address tokenAddress,
        uint256 tokenAmount
    ) external onlyOwner {
        require(
            tokenAddress != address(rewardsToken),
            "Cannot withdraw the rewardsToken"
        );
        IERC20(tokenAddress).safeTransfer(owner, tokenAmount);
        emit Recovered(tokenAddress, tokenAmount);
    }
}
```

***\*测试方法:\****

**forge test*--contracts src/test/**recoverERC20.sol** -vvvv**

```jsx
// 测试函数以演示 VulnStakeRewardsContract 中的漏洞.
function testVulnStakingRewards() public {
    // 记录操作前该合约持有的RewardTokenContract的余额。
    console.log(
        "Before rug RewardToken balance in VulnStakingRewardsContract",
        RewardTokenContract.balanceOf(address(this))
    );

    // 使用虚拟机（vm）的“prank”功能模拟一些与“alice”相关的动作。
     // “恶作剧”函数的确切行为取决于虚拟机的实现。
     // 这里的目的很可能是模拟“alice”的动作。
    vm.prank(alice);

    // 将 10,000 以太币的 RewardToken 从该合约地址转移到 VulnStakeRewardsContract。
     // 转账是通过RewardTokenContract的“转账”功能进行的。
    RewardTokenContract.transfer(
        address(VulnStakingRewardsContract),
        10000 ether
    );

    // 管理员从 VulnStakeRewardsContract 中发起 rug（提取）1,000 以太币的 RewardToken。
     // 此操作通过 VulnStakeRewardsContract 的 'recoverERC20' 函数执行。
    VulnStakingRewardsContract.recoverERC20(
        address(RewardTokenContract),
        1000 ether
    );

    // 记录操作后该合约持有的RewardTokenContract的余额。
    console.log(
        "After rug RewardToken balance in VulnStakingRewardsContract",
        RewardTokenContract.balanceOf(address(this))
    );
}
```

**红框：**没有针对奖励标记的相应检查**紫色盒子：**问题已修复。

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fd1f50846-7c7d-4703-a900-818d1aed7653%2FUntitled.png?table=block&id=9ca3f83c-7491-4679-8764-80df53ec3e6f&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)