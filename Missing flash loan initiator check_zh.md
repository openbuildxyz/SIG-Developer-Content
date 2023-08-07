# 缺少闪贷发起人检查

**名称：** 缺少快速贷款发起人检查

**描述：** 缺少闪贷发起人检查是指闪贷实施中存在潜在的安全漏洞，其中闪贷发起人未经过正确验证或检查，任何人都可以利用闪贷功能并将接收者地址设置为易受攻击的协议。

通过这样，攻击者可能会在易受攻击的协议中操纵余额、开放交易、耗尽资金或执行其他恶意操作。 这给协议及其用户的安全性和完整性带来了重大风险。

**解决办法：**

检查闪贷的发起人，如果发起人未经授权则恢复。

**参考:** https://twitter.com/ret2basic/status/1681150722434551809

https://github.com/sherlock-audit/2023-05-dodo-judging/issues/34

**SimpleBankBug 合约:**

```jsx
contract SimpleBankBug {
    using SafeERC20 for IERC20;
    IERC20 public USDa;
    LendingPool public lendingPool;

    constructor(address _lendingPoolAddress, address _asset) {
        lendingPool = LendingPool(_lendingPoolAddress);
        USDa = IERC20(_asset);
    }

    function flashLoan(
        uint256 amounts,
        address receiverAddress,
        bytes calldata data
    ) external {
        receiverAddress = address(this);

        lendingPool.flashLoan(amounts, receiverAddress, data);
    }

    function executeOperation(
        uint256 amounts,
        address receiverAddress,
        address _initiator,
        bytes calldata data
    ) external {
        /* Perform your desired logic here
        Open opsition, close opsition, drain funds, etc.
        _closetrade(...) or _opentrade(...)
        */

        // transfer all borrowed assets back to the lending pool
        IERC20(USDa).safeTransfer(address(lendingPool), amounts);
    }
}
```

**FixedSimpleBank 合约:(紫色字体**)

```jsx
contract FixedSimpleBank {
    using SafeERC20 for IERC20;
    IERC20 public USDa;
    LendingPool public lendingPool;

    constructor(address _lendingPoolAddress, address _asset) {
        lendingPool = LendingPool(_lendingPoolAddress);
        USDa = IERC20(_asset);
    }

    function flashLoan(
        uint256 amounts,
        address receiverAddress,
        bytes calldata data
    ) external {
        address receiverAddress = address(this);

        lendingPool.flashLoan(amounts, receiverAddress, data);
    }

    function executeOperation(
        uint256 amounts,
        address receiverAddress,
        address _initiator,
        bytes calldata data
    ) external {
        // Mitigation: make sure to check the initiator
        **require(_initiator == address(this), "Unauthorized");** 

        // transfer all borrowed assets back to the lending pool
        IERC20(USDa).safeTransfer(address(lendingPool), amounts);
    }
}
```

***\*测试方法:\****

**仿真测试**--contracts src/test/**Flashloan-flaw.sol** -vvvv

```jsx
// Test function to demonstrate a potential flash loan flaw.
function testFlashLoanFlaw() public {
    // Call the 'flashLoan' function of the LendingPoolContract.
    // This function initiates a flash loan of 500 ether from the lending pool.
    // The flash loan is sent to the SimpleBankBugContract, identified by its address.
    // The last argument "0x0" is an optional parameter representing additional data for the flash loan.
    // The function signature does not specify the return type, so it's assumed that the flashLoan function completes successfully.
    LendingPoolContract.flashLoan(
        500 ether,
        address(SimpleBankBugContract),
        "0x0"
    );
}
```

**红色框：**缺少闪贷发起人检查 **紫色框：**问题已修复。

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F5a43db2c-b6b1-44de-9ce2-df8ac05b0a41%2FF1SgN3KagAQq4yX.jpeg?table=block&id=5eed3d6d-98ab-422c-bbc8-4562aebcb5ce&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)