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

**缓解措施：**  
使用 OpenZeppelin 的 SafeERC20 库，将 transfer 更改为 safeTransfer。

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
// 定义一个名为 testTransfer 的函数
function testTransfer() public {
    // 使用合约(vm)方法来模拟指定地址的作恶
    vm.startPrank(0xef0DCc839c1490cEbC7209BAa11f46cfe83805ab);
    // 尝试将USDT代币从合约地址安全转移到当前合约。
    // 123是要转移的代币数量。
    // 如果合约没有足够的代币，或者startPrank中指定的地址没有授权此转移，它将抛出错误，并且当前合约中的所有状态更改都将被回滚。
    usdt.transfer(address(this), 123); //回滚
    // 停止作恶。这大概会恢复原始状态，但如果上述转帐失败，这一步将不会执行。
    vm.stopPrank();
}

// 定义公共函数testSafeTransfer
function testSafeTransfer() public {
    // 使用合约(vm)方法模拟来自指定地址的作恶
    vm.startPrank(0xef0DCc839c1490cEbC7209BAa11f46cfe83805ab);
    // 尝试将USDT代币从合约地址安全转移到本合约。
    // safeTransfer类似于transfer，但它也会检查接受者是否是一个合约。
    // 如果是合约，safeTransfer将调用接收者合约上的一个函数，以确保它已准备好接收代币。
    // 这可以防止代币被锁定在一个未准备好处理它们的合约中。
    usdt.safeTransfer(address(this), 123);
    // 停止恶作剧。与 testTransfer 类似，如果 safeTransfer 调用失败，此步骤也不会执行。
    vm.stopPrank();
}
```
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fae130f04-5cf6-4470-a349-dedc1d1d9b82%2FUntitled.png?table=block&id=994e9c1e-b643-43ec-8aee-6c8c1289ac96&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)  
**红框：**  已回滚。  

**紫色框：** 使用safeTransfer，不回滚。  

![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F1432dd60-4c60-4239-8904-cd11fd5bb63b%2FUntitled.png?table=block&id=89b3accb-ca41-4843-879a-65f7f86ba4f6&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=1840&userId=&cache=v2)
