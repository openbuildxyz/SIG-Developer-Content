# 失败后不回滚  
[Returnfalse.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Returnfalse.sol)  
**名称：** 失败后不回滚  

**描述：**
有些令牌在失败时不会回滚，而是返回false（例如 ZRX）。  
```
ZRX transfer return false:
function transfer(address _to, uint _value) returns (bool) {
//默认假定总供应量不能超过最大值(2^256 - 1)。 
if (balances[msg.sender] >= _value && balances[_to] + _value >= balances[_to]) {
balances[msg.sender] -= _value;
balances[_to] += _value;
Transfer(msg.sender, _to, _value);
return true;
} else { return false; }
}
```  
**缓解建议：**  
使用OpenZeppelin的SafeERC20库并将Transfer更改为 safeTransfer。  
**合约：**  
```
contract ContractTest is Test {
    using SafeERC20 for IERC20;
    IERC20 constant zrx = IERC20(0xE41d2489571d322189246DaFA5ebDe1F4699F498);

    function setUp() public {
        vm.createSelectFork("mainnet", 16138254);
    }

    function testTransfer() public {
        vm.startPrank(0xef0DCc839c1490cEbC7209BAa11f46cfe83805ab);
        zrx.transfer(address(this), 123); //return false, do not revert
        vm.stopPrank();
    }

    function testSafeTransferFail() public {
        vm.startPrank(0xef0DCc839c1490cEbC7209BAa11f46cfe83805ab);

        // https://github.com/foundry-rs/foundry/issues/5367 can't vm.expectRevert
        // vm.expectRevert("SafeERC20: ERC20 operation did not succeed");
        zrx.safeTransfer(address(this), 123); //revert

        vm.stopPrank();
    }

    receive() external payable {}
}
```  
**如何测试：**  
forge test --contracts src/test/Returnfalse.sol -vvvv  
```
// 定义一个公共函数testTransfer
function testTransfer() public {
    // 开始一个可能模拟恶意活动或行为的作恶
    // 或者为指定地址的测试创建某种非典型条件
    vm.startPrank(0xef0DCc839c1490cEbC7209BAa11f46cfe83805ab);
    // 尝试将123个单位的ZRX代币从合约转移到此合约。
    // 如果传输失败，“transfer”方法返回‘false’而不是回滚
    zrx.transfer(address(this), 123); //返回false而不是回滚
    // 停止作恶，这可能会使情况恢复正常
    vm.stopPrank();
}

// 定义一个公共函数testSafeTransferFail
function testSafeTransferFail() public {
    // 像前面的函数一样开始做恶
    vm.startPrank(0xef0DCc839c1490cEbC7209BAa11f46cfe83805ab);
    // 这行代码被注释掉了。它似乎在设置一个期望，即函数应该回滚是，发出消息“SafeERC20：ERC20 操作未成功”。此预期可能会用于测试。
    // 但是，URL中提到的问题表明此函数在此测试环境中不起作用
    // vm.expectRevert("SafeERC20: ERC20 operation did not succeed");
    // 尝试将123个ZRX代币从合同“安全”转移到本合同。
    // 与常规的“transfer”方法不同，如果传输失败，“safeTransfer”将回滚。
    zrx.safeTransfer(address(this), 123); //回滚
    // 停止作恶
    vm.stopPrank();
}
```  
**红色框：** 失败时不回滚。    
**紫色框：** 失败时回滚，使用safeTranser。
![Alt text](image-28.png)