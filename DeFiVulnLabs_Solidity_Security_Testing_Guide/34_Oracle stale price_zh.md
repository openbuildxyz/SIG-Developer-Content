[Oracle-stale.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Oracle-stale.sol)

# Oracle 过时价格

**名称：** Oracle 供给数据未充分验证

**描述：** Chainlink 供价最新版本数据用于从 chainlink 检索供价。 我们需要确保答案不是否定的并且价格不是陈旧的。

**解决措施：**latestAnswer 函数已弃用。 相反，请使用latestRoundData 函数检索价格并确保添加对过时数据的检查。

**参考：**

https://twitter.com/1nf0s3cpt/status/1674611468975878144

https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/94

https://code4rena.com/reports/2022-10-inverse#m-17-chainlink-oracle-data-feed-is-not-sufficiently-validated-and-can-return-stale-price

https://docs.chain.link/data-feeds/historical-data#getrounddata-return-values

**合约:**

```jsx
contract ContractTest is Test {
    AggregatorV3Interface internal priceFeed;

    function setUp() public {
        vm.createSelectFork("mainnet", 17568400);

        priceFeed = AggregatorV3Interface(
            0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419
        ); // ETH/USD
    }

    function testUnSafePrice() public {
        //Chainlink 预言机数据源未经过充分验证，可能会返回过时的价格。
        (, int256 answer, , , ) = priceFeed.latestRoundData();
        emit log_named_decimal_int("price", answer, 8);
    }

    function testSafePrice() public {
        (
            uint80 roundId,
            int256 answer,
            ,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = priceFeed.latestRoundData();
        /*
        解决办法：
         answerInRound：计算答案的回合 ID
         UpdatedAt：回合更新的时间戳
         答案：本轮的答案
        */
        require(answeredInRound >= roundId, "answer is stale");
        require(updatedAt > 0, "round is incomplete");
        require(answer > 0, "Invalid feed answer");
        emit log_named_decimal_int("price", answer, 8);
    }

    receive() external payable {}
}
```

***\*测试方法:\****

forge test --contracts src/test/**Oracle-stale.sol** -vvvv

```// Function to test the potentially unsafe retrieval of price data from a price feed
    function testUnSafePrice() public {
        // 从价格源中检索最新一轮数据
         // 未验证检索到的数据的陈旧性或完整性，如果数据过时或不完整，则可能会导致问题
        (, int256 answer, , , ) = priceFeed.latestRoundData();```
```

        // 发出包含可能过时或无效价格的日志
        emit log_named_decimal_int("price", answer, 8);
    }

```// Function to test the safe retrieval of price data from a price feed
    function testSafePrice() public {
        // 从价格源中检索最新一轮数据
         // roundId：回合的唯一标识符
         //答案：本轮价格
         // UpdatedAt：回合更新的时间戳
         //answerInRound：计算答案的回合 ID
        (
            uint80 roundId,
            int256 answer,
            ,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = priceFeed.latestRoundData();```
```

        // 通过确保计算答案时的回合 ID 大于或等于当前回合 ID 来检查答案是否过时
         // 如果没有，这将导致函数恢复
        require(answeredInRound >= roundId, "answer is stale");
    
        // 通过确保updatedAt时间戳大于0来检查回合是否完成
         // 如果没有，这将导致函数恢复
        require(updatedAt > 0, "round is incomplete");
    
        // 通过确保答案大于 0 来检查答案是否有效
         // 如果没有，这将导致函数恢复
        require(answer > 0, "Invalid feed answer");
    
        //发出包含检索到的价格的日志
        emit log_named_decimal_int("price", answer, 8);
    }

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fdcfb1287-eaae-4cf7-933e-c9d58c9c2a4a%2FUntitled.png?table=block&id=98b2dd62-064e-4594-ac18-d4b0b5c09c51&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)