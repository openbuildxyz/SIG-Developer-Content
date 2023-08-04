# Incompatibility with deflationary / fee-on-transfer tokens  
[fee-on-transfer.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/fee-on-transfer.sol)  

**Name:** STA token had a deflationary model with transfer fee of 1% charged from a recipient.

**Description:**  
The actual deposited amount might be lower than the specified depositAmount of the function parameter.

VulnVault: Incompatability with deflationary / fee-on-transfer tokens

**Mitigation:**  

Transfer the tokens first and compare pre-/after token balances to compute the actual deposited amount.

**REF:**

https://twitter.com/1nf0s3cpt/status/1671084918506684418

https://medium.com/1inch-network/balancer-hack-2020-a8f7131c980e

https://twitter.com/BlockSecTeam/status/1600442137811689473

**VulnVault  Contract:**  
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

//Mitigated vault
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
**How to Test:**

forge test --contracts src/test/**fee-on-transfer.sol**-vvvv  
```
// Function to test the vulnerability on transfer fees
    function testVulnFeeOnTransfer() public {
        // The address of Alice is set as the first address in the virtual machine
        address alice = vm.addr(1);
        
        // The address of Bob is set as the second address in the virtual machine
        address bob = vm.addr(2);
        
        // Fetches the balance of the current contract from the STAContract
        STAContract.balanceOf(address(this));
        
        // Transfers 1,000,000 tokens from the current contract to Alice
        STAContract.transfer(alice, 1000000);
        
        // Logs Alice's balance in the console, after deducting a 1% transfer fee
        console.log("Alice's STA balance:", STAContract.balanceOf(alice)); // charge 1% fee
        
        // Initiates a "prank" on Alice's account in the virtual machine (it's not clear what the prank does without additional context)
        vm.startPrank(alice);
        
        // Alice approves the Vulnerable Vault Contract to spend the maximum possible amount of tokens on her behalf
        STAContract.approve(address(VulnVaultContract), type(uint256).max);
        
        // Deposits 10,000 tokens into the Vulnerable Vault Contract from Alice's account
        VulnVaultContract.deposit(10000);

        // Logs the balance of Alice in the Vulnerable Vault Contract in the console, after deducting a 1% deposit fee
        console.log(
            "Alice deposit 10000 STA, but Alice's STA balance in VulnVaultContract:",
            VulnVaultContract.getBalance(alice)
        ); // charge 1% fee
        
        // Asserts that the balance of the Vulnerable Vault Contract in the STAContract should be equal to Alice's balance in the Vulnerable Vault Contract
        // If they are not equal, it throws an error
        assertEq(
            STAContract.balanceOf(address(VulnVaultContract)),
            VulnVaultContract.getBalance(alice)
        );
    }
```
![Alt text](image.png)
