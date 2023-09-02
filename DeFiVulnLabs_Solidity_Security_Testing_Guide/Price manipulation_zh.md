[Price_manipulation.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Price_manipulation.sol)

# 价格操纵

**名称：** 价格操纵

**描述：**balanceOf价格计算错误，getReverse可能指向一个未经精准计算的toke或资产价格。

**减轻：**

使用抗操纵预言机、chainlink、TWAP 等。

**参考：**

https://twitter.com/1nf0s3cpt/status/1673948842738487296

https://github.com/SunWeb3Sec/DeFiHackLabs/tree/main/past/2022#20221012-atk---flashloan-manipulate-price

https://github.com/SunWeb3Sec/DeFiHackLabs/tree/main/past/2022#20220807-egd-finance---flashloans--price-manipulation

https://github.com/SunWeb3Sec/DeFiHackLabs/tree/main/past/2022#20220428-deus-dao---flashloan--price-oracle-manipulation

**SimpleBank 合约样例:**

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

***\*测试方法:\****

forge test--contracts src/test/**Price_manipulation.sol** -vvvv

```jsx
// 测试潜在价格操纵漏洞的函数
    function testPrice_Manipulation() public {
        // 将 9000 以太币（这里使用以太币作为面额，但实际上指的是 USDb 代币的最小单位）从 USDbContract 转移到 SimpleBankContract
        USDbContract.transfer(address(SimpleBankContract), 9000 ether);
        
        // 将价值 1000 以太币的 USDa 代币从 USDaContract 转移到 SimplePoolContract
        USDaContract.transfer(address(SimplePoolContract), 1000 ether);
        
        // 将价值 1000 以太币的 USDb 代币从 USDbContract 转移到 SimplePoolContract
        USDbContract.transfer(address(SimplePoolContract), 1000 ether);
        
        // 从 SimplePoolContract 中检索以 USDb 表示的 USDa 的当前价格
         // 最初，假设价格为 1 USDa : 1 USDb
        SimplePoolContract.getPrice(); // 1 USDa : 1 USDb

        // 记录 USDa 价格为 1 比 1 USDb 的假设，因为池中各有 1000 个
        console.log(
            "There are 1000 USDa and USDb in the pool, so the price of USDa is 1 to 1 USDb."
        );
        
        // 发出当前 USDa 汇率的日志
        emit log_named_decimal_uint(
            "Current USDa convert rate",
            SimplePoolContract.getPrice(),
            18
        );
        
        // 记录价格操纵尝试的开始
        console.log("Start price manipulation");
        
        // 记录有关通过闪贷借入 500 美元的消息
        console.log("Borrow 500 USBa over floashloan");
        
        // 尝试通过快速贷款借入 500 USDb 来操纵价格
         // 这可能会扭曲 SimplePoolContract 中 USDa 和 USDb 的平衡，从而影响价格
        SimplePoolContract.flashLoan(500 ether, address(this), "0x0");
    }
```

**红色框：价格被操纵。**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F6e72f9da-a310-4e12-a4df-724cfe5a4cd0%2FUntitled.png?table=block&id=4982af62-42fa-4aef-ac03-b62eb5514bc9&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)