# 批准骗局  
[ApproveScam.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/ApproveScam.sol)  
**名称：**  过度宽容的批准骗局  
**描述：**  
该漏洞与ERC20代币的批准流程相关。  
在这种情况下，Alice批准Eve从Alice的账户转移无限量（type.(uint256).max）的代币。随后，Eve利用此权限将1000个代币从Alice的账户转入她的账户。  
目前大多数骗局都使用approve或setApprovalForAll来骗取您的转让权。这部分要特别小心。  

**缓解建议：**  
用户应仅批准当前操作所需的代币数量。  


**合约：**  
```
contract ContractTest is Test {
    ERC20 ERC20Contract;
    address alice = vm.addr(1);
    address eve = vm.addr(2);

    function testApproveScam() public {
        ERC20Contract = new ERC20();
        ERC20Contract.mint(1000);
        ERC20Contract.transfer(address(alice), 1000);

        vm.prank(alice);
        // 小心地向未知网站/地址授予无限量。
        // 如果您确定它来自合法网站，请不要执行批准。
        // Alice向Eve授予批准权限。
        ERC20Contract.approve(address(eve), type(uint256).max);

        console.log(
            "Before exploiting, Balance of Eve:",
            ERC20Contract.balanceOf(eve)
        );
        console.log(
            "Due to Alice granted transfer permission to Eve, now Eve can move funds from Alice"
        );
        vm.prank(eve);
        // 现在，Eve可以从Alice那里转移资金
        ERC20Contract.transferFrom(address(alice), address(eve), 1000);
        console.log(
            "After exploiting, Balance of Eve:",
            ERC20Contract.balanceOf(eve)
        );
        console.log("Exploit completed");
    }

    receive() external payable {}
}
```  
**如何测试：**  
forge test --contracts src/test/ApproveScam.sol -vvvv  
```
// 使用ERC20代币测试批准漏洞的函数
function testApproveScam() public {
    // 创建一个新的ERC20合约实例
    ERC20Contract = new ERC20();

    // 向合约余额铸造1000个新代币
    ERC20Contract.mint(1000);

    // 向Alice转账1000个代币
    ERC20Contract.transfer(address(alice), 1000);

    // Alice成为活跃的作恶者
    vm.prank(alice);

    // Alice批准Eve花费任意数量的代币。这是一个安全风险。
    ERC20Contract.approve(address(eve), type(uint256).max);

    // 记录漏洞攻击前Eve的余额
    console.log(
        "Before exploiting, Balance of Eve:",
        ERC20Contract.balanceOf(eve)
    );

    // 记录警告
    console.log(
        "Due to Alice granted transfer permission to Eve, now Eve can move funds from Alice"
    );

    // Eve成为活跃的作恶者
    vm.prank(eve);

    // Eve将Alice的1000个代币转移给自己
    ERC20Contract.transferFrom(address(alice), address(eve), 1000);

    // 记录漏洞攻击后Eve的余额
    console.log(
        "After exploiting, Balance of Eve:",
        ERC20Contract.balanceOf(eve)
    );

    // 记录“漏洞完成”。
    console.log("Exploit completed");
}
```
**红框：** 由于Alice授予Eve批准，因此转移了Alice的资金。 
![Alt text](image-18.png)
