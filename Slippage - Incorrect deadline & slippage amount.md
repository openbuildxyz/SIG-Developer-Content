# Slippage - Incorrect deadline & slippage amount

**Name:** Slippage - Incorrect deadline & slippage amount

**Description:** Slippage: Slippage is the difference between the expected price of a trade and the price at which the trade is executed. If hardcoded to 0, user will accept a minimum amount of 0 output tokens from the swap.

Deadline: The function sets the deadline to the maximum uint256 value, which means the transaction can be executed at any time.

If slippage is set to 0 and there is no deadline, users might potentially lose all their tokens.

**Mitigation:** Allow the user to specify the slippage & deadline value themselves.

**REF:**

https://twitter.com/1nf0s3cpt/status/1676118132992405505

**Contract:**

```jsx
contract ContractTest is Test {
    address UNISWAP_ROUTER = 0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D; // Uniswap Router address on Ethereum Mainnet
    IWETH WETH = IWETH(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
    address USDT = 0xdAC17F958D2ee523a2206206994597C13D831ec7;

    function setUp() public {
        vm.createSelectFork("mainnet", 17568400);
    }

    function testswapTokensWithMaxDeadline() external payable {
        WETH.approve(address(UNISWAP_ROUTER), type(uint256).max);
        WETH.deposit{value: 1 ether}();

        uint256 amountIn = 1 ether;
        uint256 amountOutMin = 0;
        //uint256 amountOutMin = 1867363899; //1867363899 INSUFFICIENT_OUTPUT_AMOUNT
        // Path for swapping ETH to USDT
        address[] memory path = new address[](2);
        path[0] = address(WETH); // WETH (Wrapped Ether)
        path[1] = USDT; // USDT (Tether)

        // No Effective Expiration Deadline
        // The function sets the deadline to the maximum uint256 value, which means the transaction can be executed at any time,
        // possibly under unfavorable market conditions.
        IUniswapV2Router02(UNISWAP_ROUTER).swapExactTokensForTokens(
            amountIn,
            amountOutMin,
            path,
            address(this),
            type(uint256).max // Setting deadline to max value
        );

        console.log("USDT", IERC20(USDT).balanceOf(address(this)));
    }
```

***\*How to Test:\****

**forge test** --contracts src/test/**Slippage-deadline.sol**-vvvv

```// Function to test swapping tokens with the maximum expiration deadline (no effective deadline).
function testswapTokensWithMaxDeadline() external payable {
    // Approve the maximum allowance for UNISWAP_ROUTER to spend WETH tokens.
    WETH.approve(address(UNISWAP_ROUTER), type(uint256).max);```
```

    // Deposit 1 Ether (ETH) into WETH (Wrapped Ether) to get WETH tokens.
    WETH.deposit{value: 1 ether}();
    
    // Define the amount of input tokens (WETH) to swap.
    uint256 amountIn = 1 ether;
    
    // Define the minimum amount of output tokens (USDT) accepted after the swap.
    // In this case, 'amountOutMin' is set to 0, which means the swap will not be reverted even if the output amount is 0.
    uint256 amountOutMin = 0;
    
    // Define the path for swapping ETH to USDT.
    // The path contains two addresses: WETH (Wrapped Ether) and USDT (Tether).
    address[] memory path = new address[](2);
    path[0] = address(WETH); // WETH (Wrapped Ether)
    path[1] = USDT; // USDT (Tether)
    
    // Perform the token swap using the Uniswap V2 Router 02.
    // The function used is 'swapExactTokensForTokens', which means the contract will receive exactly the specified
    // 'amountOutMin' or revert the transaction if it cannot receive at least 'amountOutMin' USDT tokens.
    // The 'type(uint256).max' value as the deadline allows the transaction to be executed at any time.
    IUniswapV2Router02(UNISWAP_ROUTER).swapExactTokensForTokens(
        amountIn,          // The exact amount of input tokens (WETH) to be swapped.
        amountOutMin,      // The minimum amount of output tokens (USDT) accepted after the swap.
        path,              // The path representing the token swap route.
        address(this),     // The address to receive the output tokens (USDT).
        type(uint256).max  // Setting the deadline to the maximum uint256 value (no effective deadline).
    );
    
    // Log the balance of USDT tokens held by this contract after the swap.
    console.log("USDT", IERC20(USDT).balanceOf(address(this)));
}

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F1693e485-fb66-4976-8da6-4c799151e6ca%2FUntitled.png?table=block&id=45929574-a4fd-4d70-809d-21e67bbccf92&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)