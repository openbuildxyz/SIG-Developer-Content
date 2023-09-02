[Slippage-deadline.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Slippage-deadline.sol)

# 滑点 - 截止日期和滑点金额不正确

**名称：** 滑点 - 截止日期和滑点金额不正确

**说明：** 滑点：滑点是交易的预期价格与交易执行价格之间的差额。 如果硬编码为 0，用户将从交换中接受最少 0 个输出代币。

Deadline：该函数将截止时间设置为最大uint256值，这意味着交易可以在任何时间执行。

如果滑点设置为 0 并且没有截止日期，用户可能会丢失所有代币。

**解决措施：** 允许用户自己指定滑点和截止日期值。

**参考：**

https://twitter.com/1nf0s3cpt/status/1676118132992405505

**合约样例:**

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
        // ETH兑换USDT的路径
        address[] memory path = new address[](2);
        path[0] = address(WETH); // WETH (Wrapped Ether)
        path[1] = USDT; // USDT (Tether)

        // 无有效到期期限
         // 该函数将deadline设置为最大uint256值，这意味着交易可以随时执行，
         // 可能在不利的市场条件下。
        IUniswapV2Router02(UNISWAP_ROUTER).swapExactTokensForTokens(
            amountIn,
            amountOutMin,
            path,
            address(this),
            type(uint256).max // 将截止日期设置为最大值
        );

        console.log("USDT", IERC20(USDT).balanceOf(address(this)));
    }
```

***\*测试方法:\****

**forge test** --contracts src/test/**Slippage-deadline.sol**-vvvv

```
function testswapTokensWithMaxDeadline() external payable {
    // 批准 UNISWAP_ROUTER 花费 WETH 代币的最大限额。
    WETH.approve(address(UNISWAP_ROUTER), type(uint256).max);
```

    // 将1个以太币（ETH）存入WETH（Wrapped Ether）以获得WETH代币。
    WETH.deposit{value: 1 ether}();
    
    // 定义要交换的输入代币 (WETH) 的数量。
    uint256 amountIn = 1 ether;
    
    // 定义交换后接受的最小输出代币（USDT）数量。
    // 在这种情况下，'amountOutMin'设置为0，这意味着即使输出金额为0，交换也不会被恢复。
    uint256 amountOutMin = 0;
    
    // 定义ETH兑换USDT的路径。
    // 该路径包含两个地址：WETH（Wrapped Ether）和USDT（Tether）。
    address[] memory path = new address[](2);
    path[0] = address(WETH); // WETH (Wrapped Ether)
    path[1] = USDT; // USDT (Tether)
    
    // 使用 Uniswap V2 Router 02 执行代币交换。
    // 使用的函数是'swapExactTokensForTokens'，这意味着合约将准确接收指定的
    // 'amountOutMin' 或在无法接收至少 'amountOutMin' USDT 代币时恢复交易。
    // 以 'type(uint256).max' 值作为截止时间，允许交易随时执行。
    IUniswapV2Router02(UNISWAP_ROUTER).swapExactTokensForTokens(
        amountIn,          // 要交换的输入代币 (WETH) 的确切数量。
        amountOutMin,      // 兑换后接受的最小输出代币（USDT）数量。
        path,              // 代表代币交换路由的路径。
        address(this),     // 接收输出代币（USDT）的地址。
        type(uint256).max  // 将截止日期设置为最大 uint256 值（无有效截止日期）。
    );
    
    // 记录掉期后该合约持有的USDT代币余额。
    console.log("USDT", IERC20(USDT).balanceOf(address(this)));
}

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F1693e485-fb66-4976-8da6-4c799151e6ca%2FUntitled.png?table=block&id=45929574-a4fd-4d70-809d-21e67bbccf92&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)