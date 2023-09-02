[Flashloan-flaw.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Flashloan-flaw.sol)

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
        /* 在这里执行你想要的逻辑
        Open opsition, close opsition, drain funds, etc.
        _closetrade(...) or _opentrade(...)
        */

        // 将所有借入资产转回出借池
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
        // 措施：确保检查启动器
        **require(_initiator == address(this), "Unauthorized");** 

        // 将所有借入资产转回出借池
        IERC20(USDa).safeTransfer(address(lendingPool), amounts);
    }
}
```

***\*测试方法:\****

**forge test**--contracts src/test/**Flashloan-flaw.sol** -vvvv

```jsx
// 测试函数以演示潜在的闪贷缺陷。
function testFlashLoanFlaw() public {
    // 调用 LendingPoolContract 的 'flashLoan' 函数。
     // 该函数从借贷池发起 500 以太币的闪电贷。
     // 闪电贷被发送到 SimpleBankBugContract，由其地址标识。
     // 最后一个参数“0x0”是可选参数，表示闪贷的附加数据。
     // 函数签名没有指定返回类型，因此假设 flashLoan 函数成功完成。
    LendingPoolContract.flashLoan(
        500 ether,
        address(SimpleBankBugContract),
        "0x0"
    );
}
```

**红色框：**缺少闪贷发起人检查 **紫色框：**问题已修复。

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F5a43db2c-b6b1-44de-9ce2-df8ac05b0a41%2FF1SgN3KagAQq4yX.jpeg?table=block&id=5eed3d6d-98ab-422c-bbc8-4562aebcb5ce&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)