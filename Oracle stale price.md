# Oracle stale price

**Name:** Oracle data feed is insufficiently validated

**Description:** Chainlink price feed latestRoundData is used to retrieve price feed from chainlink. We need to makes sure that the answer is not negative and price is not stale.

**Mitigation:** latestAnswer function is deprecated. Instead, use the latestRoundData function to retrieve the price and make sure to add checks for stale data.

**REF:**

https://twitter.com/1nf0s3cpt/status/1674611468975878144

https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/94

https://code4rena.com/reports/2022-10-inverse#m-17-chainlink-oracle-data-feed-is-not-sufficiently-validated-and-can-return-stale-price

https://docs.chain.link/data-feeds/historical-data#getrounddata-return-values

**Contract:**

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
        //Chainlink oracle data feed is not sufficiently validated and can return stale price.
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
        Mitigation:
        answeredInRound: The round ID in which the answer was computed
        updatedAt: Timestamp of when the round was updated
        answer: The answer for this round
        */
        require(answeredInRound >= roundId, "answer is stale");
        require(updatedAt > 0, "round is incomplete");
        require(answer > 0, "Invalid feed answer");
        emit log_named_decimal_int("price", answer, 8);
    }

    receive() external payable {}
}
```

***\*How to Test:\****

forge test --contracts src/test/**Oracle-stale.sol** -vvvv

```// Function to test the potentially unsafe retrieval of price data from a price feed
    function testUnSafePrice() public {
        // Retrieves the latest round data from the price feed
        // The data retrieved is not validated for staleness or completeness, potentially leading to issues if the data is outdated or incomplete
        (, int256 answer, , , ) = priceFeed.latestRoundData();```
```

        // Emits a log with the potentially stale or invalid price
        emit log_named_decimal_int("price", answer, 8);
    }

```// Function to test the safe retrieval of price data from a price feed
    function testSafePrice() public {
        // Retrieves the latest round data from the price feed
        // roundId: the unique identifier for the round
        // answer: the price for this round
        // updatedAt: the timestamp of when the round was updated
        // answeredInRound: the round ID in which the answer was computed
        (
            uint80 roundId,
            int256 answer,
            ,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = priceFeed.latestRoundData();```
```

        // Check that the answer is not stale by ensuring that the round ID of when the answer was computed is greater than or equal to the current round ID
        // If not, this will cause the function to revert
        require(answeredInRound >= roundId, "answer is stale");
    
        // Check that the round is complete by ensuring that the updatedAt timestamp is greater than 0
        // If not, this will cause the function to revert
        require(updatedAt > 0, "round is incomplete");
    
        // Check that the answer is valid by ensuring that it's greater than 0
        // If not, this will cause the function to revert
        require(answer > 0, "Invalid feed answer");
    
        // Emits a log with the retrieved price
        emit log_named_decimal_int("price", answer, 8);
    }

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fdcfb1287-eaae-4cf7-933e-c9d58c9c2a4a%2FUntitled.png?table=block&id=98b2dd62-064e-4594-ac18-d4b0b5c09c51&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)