# Return vs break

**Name:** Use of return in inner loop iteration leads to unintended termination.

**Description:** BankContractBug's addBanks function exhibits an incorrect usage of the return statement within a loop iteration, resulting in unintended termination of the loop. The return statement is placed inside the inner loop, causing premature exit from the function before completing the iteration over all bank addresses.

**Mitigation:**

Use break instead of return

**REF:**

https://twitter.com/1nf0s3cpt/status/1678596730865221632

https://github.com/code-423n4/2022-03-lifinance-findings/issues/34

https://solidity-by-example.org/loop/

**BankContractBug Contract:**

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

**FixedBank Contract: （Purple Ｗords)**

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

***\*How to Test:\****

forge test --contracts src/test/***\*return-break.sol\**** -vvvv

```jsx
// Test function to check for a return bug in the BankContractBugContract.
function testReturnBug() public {
    // Create dynamic arrays to store bank addresses and bank names.
    address[] memory bankAddresses = new address[](3);
    string[] memory bankNames = new string[](3);

    // Bank account 1
    // Set the address and name for the first bank account.
    bankAddresses[0] = address(1);
    bankNames[0] = "ABC Bank";

    // Bank account 2
    // Set the address and name for the second bank account.
    bankAddresses[1] = address(2);
    bankNames[1] = "XYZ Bank";

    // Bank account 3
    // Set the address and name for the third bank account.
    bankAddresses[2] = address(3);
    bankNames[2] = "Global Bank";

    // Call the 'addBanks' function of the BankContractBugContract and pass the bank addresses and names.
    // This function is expected to add the banks to the contract's storage.
    BankContractBugContract.addBanks(bankAddresses, bankNames);

    // Call the 'getBankCount' function of the BankContractBugContract.
    // This function is expected to return the total number of banks added to the contract.
    BankContractBugContract.getBankCount();
}
```

**Red box:**

BankContractBug's addBanks function exhibits an incorrect usage of the return statement within a loop iteration, resulting in unintended termination of the loop. **Purple box:** issue fixed.

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F7f059d05-d97f-409a-9388-46e9c1bcdb93%2FUntitled.png?table=block&id=3b5da020-5126-44f5-b603-e5d62fa07ec0&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)