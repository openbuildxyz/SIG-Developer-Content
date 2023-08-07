# Missing flash loan initiator check

**Name:** Missing flash loan initiator check

**Description:** Missing flash loan initiator check refers to a potential security vulnerability in a flash loan implementation where the initiator of the flash loan is not properly verified or checked, anyone could exploit the flash loan functionality and set the receiver address to a vulnerable protocol.

By doing so, an attacker could potentially manipulate balances, open trades, drain funds, or carry out other malicious actions within the vulnerable protocol. This poses significant risks to the security and integrity of the protocol and its users.

**Mitigation:**

Check the initiator of the flash loan and revert if the initiator is not authorized.

**REF:** https://twitter.com/ret2basic/status/1681150722434551809

https://github.com/sherlock-audit/2023-05-dodo-judging/issues/34

**SimpleBankBug Contract:**

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

**FixedSimpleBank Contract:(Purple** Words)

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

***\*How to Test:\****

forge test --contracts src/test/**Flashloan-flaw.sol** -vvvv

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

**Red box:** Missing flash loan initiator check **Purple box:** issue fixed.

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F5a43db2c-b6b1-44de-9ce2-df8ac05b0a41%2FF1SgN3KagAQq4yX.jpeg?table=block&id=5eed3d6d-98ab-422c-bbc8-4562aebcb5ce&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)