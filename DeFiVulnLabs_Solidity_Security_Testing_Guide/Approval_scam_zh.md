# 授权骗局  
[ApproveScam.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/ApproveScam.sol)  


**名称：**  权限过大的授权漏洞  

**描述：**  
该漏洞与ERC20代币的授权流程相关。  
在这种情况下，Alice授予Eve可以从自己的账户转移任意数量（type.(uint256).max）的代币的权限。  
随后，Eve利用此权限将1000个代币从Alice的账户转入自己账户。   

目前大多数骗局都使用approve或setApprovalForAll来骗取你的transfer权限。一定要特别小心这类骗局。  

**解决方法：**  
用户应仅授权当前交易所需的代币数量。  


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
        // 向未知网站/地址授权无限量代币时要小心。
        // 如果您确定它来自非法网站，请不要授权。
        // Alice向Eve授权。
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
// 一个使用ERC20代币测试授权漏洞的函数
function testApproveScam() public {
    // 创建一个新的ERC20合约实例
    ERC20Contract = new ERC20();

    // 合约铸造1000个新代币，现在合约余额新增了1000
    ERC20Contract.mint(1000);

    // 向Alice转账1000个代币
    ERC20Contract.transfer(address(alice), 1000);

    // 设置alice为approve()的调用者
    vm.prank(alice);

    // Alice授权Eve使用她的任意数量的代币。这是一个安全风险。
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

    // 设置Eve设置alice为transferFrom()的调用者
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
**红框：** 由于Alice给Eve授权了，所以Eve转移了Alice的资金。 
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F12373b11-e2f8-4102-a483-81694a89fc20%2FUntitled.png?table=block&id=3f9b98bb-6152-4571-a20a-a0c4e2304b76&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)
