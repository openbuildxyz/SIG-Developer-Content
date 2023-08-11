# 通货紧缩/转账费用代币不兼容  
[fee-on-transfer.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/fee-on-transfer.sol)  

**名称：** STA 代币采用通缩模型，向收款人收取 1% 的转账手续费。  

**描述：**  
实际存入金额可能低于函数参数depositAmount指定的金额。  

VulnVault：通货紧缩/转账手续费代币不兼容   

**缓解建议：**  
首先转移代币，然后比较前后的代币余额，以计算实际存入金额。  

**参考：**  
https://twitter.com/1nf0s3cpt/status/1671084918506684418  
https://medium.com/1inch-network/balancer-hack-2020-a8f7131c980e  
https://twitter.com/BlockSecTeam/status/1600442137811689473  


**VulnVault合约：**  
```
contract VulnVault {
    mapping(address => uint256) private balances;
    uint256 private fee;
    IERC20 private token;

    event Deposit(address indexed depositor, uint256 amount);
    event Withdrawal(address indexed recipient, uint256 amount);

    constructor(address _tokenAddress) {
        token = IERC20(_tokenAddress);
    }

    function deposit(uint256 amount) external {
        require(amount > 0, "Deposit amount must be greater than zero");

        token.transferFrom(msg.sender, address(this), amount);
        balances[msg.sender] += amount;
        emit Deposit(msg.sender, amount);
    }

    function withdraw(uint256 amount) external {
        require(amount > 0, "Withdrawal amount must be greater than zero");
        require(amount <= balances[msg.sender], "Insufficient balance");

        balances[msg.sender] -= amount;
        token.transfer(msg.sender, amount);
        emit Withdrawal(msg.sender, amount);
    }

    function getBalance(address account) external view returns (uint256) {
        return balances[account];
    }
}

//缓解金库
contract Vault {
    mapping(address => uint256) private balances;
    uint256 private fee;
    IERC20 private token;

    event Deposit(address indexed depositor, uint256 amount);
    event Withdrawal(address indexed recipient, uint256 amount);

    constructor(address _tokenAddress) {
        token = IERC20(_tokenAddress);
    }

    function deposit(uint256 amount) external {
        require(amount > 0, "Deposit amount must be greater than zero");

        uint256 balanceBefore = token.balanceOf(address(this));

        token.transferFrom(msg.sender, address(this), amount);

        uint256 balanceAfter = token.balanceOf(address(this));
        uint256 actualDepositAmount = balanceAfter - balanceBefore;

        balances[msg.sender] += actualDepositAmount;
        emit Deposit(msg.sender, actualDepositAmount);
    }

    function withdraw(uint256 amount) external {
        require(amount > 0, "Withdrawal amount must be greater than zero");
        require(amount <= balances[msg.sender], "Insufficient balance");

        balances[msg.sender] -= amount;
        token.transfer(msg.sender, amount);
        emit Withdrawal(msg.sender, amount);
    }

    function getBalance(address account) external view returns (uint256) {
        return balances[account];
    }
}
```  
**如何测试：**  
forge test --contracts src/test/fee-on-transfer.sol-vvvv  
```
// 测试转账手续费漏洞的函数
    function testVulnFeeOnTransfer() public {
        // Alice的地址设置为虚拟机中的第一个地址
        address alice = vm.addr(1);
        
        // Bob的地址设置为虚拟机中的第二个地址
        address bob = vm.addr(2);
        
        // 从STAContract获取当前合约的余额
        STAContract.balanceOf(address(this));
        
        //将1000000个代币从当前合约转移到Alice
        STAContract.transfer(alice, 1000000);
        
        // 在扣除1%的转账费后，在控制台中记录Alice的余额
        console.log("Alice's STA balance:", STAContract.balanceOf(alice)); // 收取1%的费用
        
        // 在虚拟机中的Alice帐户上发起“恶作剧”（如果没有其他上下文，则不清楚作恶的做了什么）
        vm.startPrank(alice);
        
        // Alice授权易受攻击的Vault合约代表她花费尽可能多的代币
        STAContract.approve(address(VulnVaultContract), type(uint256).max);
        
        //从Alice的账户将10000个代币存入易受攻击的Vault合约
        VulnVaultContract.deposit(10000);

        // 在扣除1%的存款费用后，在控制台的中记录，易受攻击的Vault合约中Alice的余额
        console.log(
            "Alice deposit 10000 STA, but Alice's STA balance in VulnVaultContract:",
            VulnVaultContract.getBalance(alice)
        ); // 收取1%的费用
        
        // 判断STAContract中易受攻击的Vault合约的余额是否等于Alice在易受攻击的Vault合约中的余额
        // 如果它们不相等，则会抛出错误
        assertEq(
            STAContract.balanceOf(address(VulnVaultContract)),
            VulnVaultContract.getBalance(alice)
        );
    }
```  
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F2d48717c-91f7-4c35-90ea-a4502725d089%2FUntitled.png?table=block&id=eea7bc3c-af25-498a-adf9-546cb1ea625d&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=1290&userId=&cache=v2)
