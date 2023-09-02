# 失败时没有回滚  
[Returnfalse.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Returnfalse.sol)  

**名称：** 失败时没有回滚  

**描述：**  
有些代币在失败时不会回滚，而是返回false（例如 ZRX）。  
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
**缓解措施：**  
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
    // 尝试将123个单位的ZRX代币从合约转移到当前合约。
    // 如果转帐失败，“transfer”方法返回‘false’而不是回滚
    zrx.transfer(address(this), 123); //返回false，不回滚
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
    // 尝试将123个ZRX代币从合约“安全”转移到本合约。
    // 与常规的“transfer”方法不同，如果转帐失败，“safeTransfer”将回滚。
    zrx.safeTransfer(address(this), 123); //回滚
    // 停止作恶
    vm.stopPrank();
}
```  
**红色框：** 失败时不回滚。   

**紫色框：** 失败时回滚，使用safeTranser。
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F70a109c0-22ec-4d60-9a60-494b7a5aae5c%2FUntitled.png?table=block&id=af8e2066-e87c-4f97-9fe0-774e6c2888d8&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=1840&userId=&cache=v2)