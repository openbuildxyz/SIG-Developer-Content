[return-break.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/return-break.sol)

# 返回与中断

**名称：** 在内循环迭代中使用 return 会导致意外终止。

**描述：** BankContractBug 的 addBanks 函数在循环迭代中错误地使用了 return 语句，导致循环意外终止。 return 语句放置在内部循环内部，导致在完成对所有存储体地址的迭代之前从函数中过早退出。

**解决办法：**

使用break代替return

**参考：**

https://twitter.com/1nf0s3cpt/status/1678596730865221632

https://github.com/code-423n4/2022-03-lifinance-findings/issues/34

https://solidity-by-example.org/loop/

**漏洞合约样例:**

```jsx
contract BankContractBug {
    struct Bank {
        address bankAddress;
        string bankName;
    }

    Bank[] public banks;

    function addBanks(
        address[] memory bankAddresses,
        string[] memory bankNames
    ) public {
        require(
            bankAddresses.length == bankNames.length,
            "Input arrays must have the same length."
        );

        for (uint i = 0; i < bankAddresses.length; i++) {
            if (bankAddresses[i] == address(0)) {
                continue;
            }

            // i++ is not executed when return is executed
            for (i = 0; i < bankAddresses.length; i++) {
                banks.push(Bank(bankAddresses[i], bankNames[i]));
                return;
            }
        }
    }

    function getBankCount() public view returns (uint) {
        return banks.length;
    }
}
```

**已修复合约样例： （紫色字体)**

```jsx
contract FixedBank {
    struct Bank {
        address bankAddress;
        string bankName;
    }

    Bank[] public banks;

    function addBanks(
        address[] memory bankAddresses,
        string[] memory bankNames
    ) public {
        require(
            bankAddresses.length == bankNames.length,
            "Input arrays must have the same length."
        );

        for (uint i = 0; i < bankAddresses.length; i++) {
            if (bankAddresses[i] == address(0)) {
                continue;
            }

            for (uint i = 0; i < bankAddresses.length; i++) {
                banks.push(Bank(bankAddresses[i], bankNames[i]));
                break; // Correct usage of break to terminate the inner loop
            }
        }
    }

    function getBankCount() public view returns (uint) {
        return banks.length;
    }
}
```

***\*测试方法:\****

**forge test --contracts src/test/***\*return-break.sol\**** -vvvv**

```jsx
// 测试函数以检查 BankContractBugContract 中的返回错误。
function testReturnBug() public {
    // 创建动态数组来存储银行地址和银行名称。
    address[] memory bankAddresses = new address[](3);
    string[] memory bankNames = new string[](3);

    //银行账户1
     // 设置第一个银行帐户的地址和名称。
    bankAddresses[0] = address(1);
    bankNames[0] = "ABC Bank";

    // 银行账户2
     // 设置第二个银行帐户的地址和名称。
    bankAddresses[1] = address(2);
    bankNames[1] = "XYZ Bank";

    //银行账户3
     // 设置第三个银行帐户的地址和名称。
    bankAddresses[2] = address(3);
    bankNames[2] = "Global Bank";

    //调用 BankContractBugContract 的 'addBanks' 函数并传递银行地址和名称。
     // 该函数预计会将银行添加到合约的存储中。
    BankContractBugContract.addBanks(bankAddresses, bankNames);

    // 调用 BankContractBugContract 的“getBankCount”函数。
     // 该函数预计返回添加到合约中的银行总数。
    BankContractBugContract.getBankCount();
}
```

**红色框：**BankContractBug 的 addBanks 函数在循环迭代中错误地使用了 return 语句，导致循环意外终止。 **紫色框：**问题已修复。

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F7f059d05-d97f-409a-9388-46e9c1bcdb93%2FUntitled.png?table=block&id=3b5da020-5126-44f5-b603-e5d62fa07ec0&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)