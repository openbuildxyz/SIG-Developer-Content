# 未检查的返回值  
[Returnvalue.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Returnvalue.sol)  
**名称：** 未检查的返回值  
**描述：**  
**EIP20标准：**  
返回一个布尔值，来表示操作是否成功。  
function transfer(address to, uint256 amount) external returns (bool);  

USDT没有正确实现EIP20标准，因此使用正确的EIP20函数签名调用这些函数将始终会回滚。  
function transfer(address to, uint256 amount) external;    
**ERC20转账:** 
```
function transfer(address to, uint256 amount) public virtual returns (bool) {
address owner = _msgSender();
_transfer(owner, to, amount);
return true;
``` 
**USDT转账无返回值:**  
```
function transfer(address _to, uint _value) public onlyPayloadSize(2 * 32) {
...
}
Transfer(msg.sender, _to, sendAmount);
}
``` 

**缓解建议：**  
使用OpenZeppelin的SafeERC20库并将Transfer更改为safeTransfer。  

**参考：**  
https://twitter.com/1nf0s3cpt/status/1600868995007410176  


**合约：**  
```
interface USDT {
    function transfer(address to, uint256 value) external;

    function balanceOf(address account) external view returns (uint256);

    function approve(address spender, uint256 value) external;
}

contract ContractTest is Test {
    using SafeERC20 for IERC20;
    IERC20 constant usdt = IERC20(0xdAC17F958D2ee523a2206206994597C13D831ec7);

    function setUp() public {
        vm.createSelectFork("mainnet", 16138254);
    }

    function testTransfer() public {
        vm.startPrank(0xef0DCc839c1490cEbC7209BAa11f46cfe83805ab);
        usdt.transfer(address(this), 123); //revert
        vm.stopPrank();
    }

    function testSafeTransfer() public {
        vm.startPrank(0xef0DCc839c1490cEbC7209BAa11f46cfe83805ab);
        usdt.safeTransfer(address(this), 123);
        vm.stopPrank();
    }

    receive() external payable {}
}
```  
**如何测试：**  
forge test --contracts src/test/Returnvalue.sol -vvvv  
```
// 定义公共函数testTransfer
function testTransfer() public {
    // 使用合约(vm)方法模拟来自指定地址的作恶
    vm.startPrank(0xef0DCc839c1490cEbC7209BAa11f46cfe83805ab);
    // 尝试将USDT代币从合约地址安全转移到本合约。
    // 123是要转移的代币数量。
    // 如果合约没有足够的代币，或者startPrank中指定的地址没有授权此转移，它将抛出错误，并且当前合约中的所有状态更改都将被还原。
    usdt.transfer(address(this), 123); //回滚
    // 停止作恶。这大概会恢复原始状态，但如果上述转移失败，则不会执行。
    vm.stopPrank();
}

// 定义公共函数testSafeTransfer
function testSafeTransfer() public {
    // 使用合约(vm)方法模拟来自指定地址的作恶
    vm.startPrank(0xef0DCc839c1490cEbC7209BAa11f46cfe83805ab);
    // 尝试将USDT代币从合约地址安全转移到本合约。
    // safeTransfer类似于transfer，但它也会检查收件人是否是合约。
    // 如果是，safeTransfer将在接收方合约上调用一个函数，以确保它准备好接收代币。
    // 这可以防止代币被锁定在未准备好处理它们的合约中。
    usdt.safeTransfer(address(this), 123);
    // 停止做 恶。与testTransfer一样，如果安全转移调用失败，则不会执行此操作。
    vm.stopPrank();
}
```
![Alt text](image-26.png)  
**红框：**  已回滚。  
**紫色框：** 使用safeTransfer，不回滚。
![Alt text](image-27.png)
