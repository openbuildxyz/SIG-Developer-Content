# Incorrect implementation of the recoverERC20() function in the StakingRewards

**Name:** Incorrect implementation of the recoverERC20() function in the StakingRewards

**Description:** The recoverERC20() function in StakingRewards.sol can potentially serve as a backdoor for the owner to retrieve rewardsToken. There is no corresponding check against the rewardsToken. This creates an administrative privilege where the owner can sweep the rewards tokens, potentially using it as a means to exploit depositors. It's similar to a forked issue if you forked vulnerable code.

**Mitigation:**

disallowing recovery of the rewardToken within the recoverErc20 function

**REF:**

https://twitter.com/1nf0s3cpt/status/1680806251482189824

https://github.com/code-423n4/2022-02-concur-findings/issues/210

https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/49

https://github.com/code-423n4/2022-10-paladin-findings/issues/40

https://blog.openzeppelin.com/across-token-and-token-distributor-audit#anyone-can-prevent-stakers-from-getting-their-rewards

**VulnStakingRewards Contract:**

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

Bug f**ixed Contract:(Purple** words)

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

***\*How to Test:\****

forge test --contracts src/test/**recoverERC20.sol** -vvvv

```jsx
// Test function to demonstrate a vulnerability in the VulnStakingRewardsContract.
function testVulnStakingRewards() public {
    // Log the balance of the RewardTokenContract held by this contract before the operations.
    console.log(
        "Before rug RewardToken balance in VulnStakingRewardsContract",
        RewardTokenContract.balanceOf(address(this))
    );

    // Simulate some action related to 'alice' using the 'prank' function of the virtual machine (vm).
    // The exact behavior of the 'prank' function depends on the virtual machine's implementation.
    // The purpose here is likely to simulate an action by 'alice'.
    vm.prank(alice);

    // Transfer 10,000 ether of RewardToken from address of this contract to the VulnStakingRewardsContract.
    // The transfer is conducted through the 'transfer' function of the RewardTokenContract.
    RewardTokenContract.transfer(
        address(VulnStakingRewardsContract),
        10000 ether
    );

    // Admin initiates a rug (withdraws) 1,000 ether of RewardToken from the VulnStakingRewardsContract.
    // This operation is executed through the 'recoverERC20' function of the VulnStakingRewardsContract.
    VulnStakingRewardsContract.recoverERC20(
        address(RewardTokenContract),
        1000 ether
    );

    // Log the balance of the RewardTokenContract held by this contract after the operations.
    console.log(
        "After rug RewardToken balance in VulnStakingRewardsContract",
        RewardTokenContract.balanceOf(address(this))
    );
}
```

**Red box:** There is no corresponding check against the rewardsToken.. **Purple box:** issue fixed.

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fd1f50846-7c7d-4703-a900-818d1aed7653%2FUntitled.png?table=block&id=9ca3f83c-7491-4679-8764-80df53ec3e6f&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)