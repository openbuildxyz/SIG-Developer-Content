# Divide before multiply 
[Divmultiply.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Divmultiply.sol)  
**Name:** Precision Issues - Divide before multiply

**Description:**  
The contracts demonstrate a common issue when performing division operations in Solidity,
as Solidity doesn't support floating-point numbers. The order of operations can affect the result due to integer truncation.

In the Miscalculation contract, the function price performs the division before the
multiplication (price / 100) * discount. Due to the fact that Solidity truncates integers
when dividing, the result of price / 100 will be 0 if the price is less than 100.
This causes the result of the multiplication to be 0 as well.

On the other hand, in the Calculation contract, the function price performs the multiplication
before the division (price * discount) / 100. This way, the result will be correct as the multiplication
doesn't get truncated, only the final result does.

**Mitigation:**  

Always perform multiplication before division to avoid losing precision.

**REF:**

https://twitter.com/1nf0s3cpt/status/1599774264437395461

https://blog.solidityscan.com/precision-loss-in-arithmetic-operations-8729aea20be9

**Miscalculation Contract:**  
```
contract Miscalculation {
    function price(
        uint256 price,
        uint256 discount
    ) public pure returns (uint256) {
        return (price / 100) * discount; // wrong calculation
    }
}
```
**How to Test:**

forge test --contracts src/test/**Divmultiply.sol**-vvv  
```
// This function is public, which means it can be called by anyone or any other contract.
function testMiscalculation() public {

    // Initialize a new instance of the MiscalculationContract.
    MiscalculationContract = new Miscalculation();
    // Log a string message to the console to indicate the test on MiscalculationContract is starting.
    console.log("Perform Miscalculation Contract");
    // Log the scenario details.
    console.log("Scenario: DeFi store 10% off now, Then we buy 1 item price: $80.");
    // Call the price method on the MiscalculationContract with two arguments, 80 and 90, and log the return value.
    console.log("Subtract the discount, get the sale price:", MiscalculationContract.price(80, 90));
    // Log the explanation of the wrong calculation.
    console.log("Solidity doesn't do decimals, so dividing before multiplying will round to zero. 0.8*90=0");
    // Log a line to separate the two different scenarios.
    console.log("---------------------------------------------------------");
    // Initialize a new instance of the CalculationContract.
    CalculationContract = new Calculation();
    // Log a string message to the console to indicate the test on CalculationContract is starting.
    console.log("Perform Correct calculation Contract");
    // Log the scenario details.
    console.log("Scenario: DeFi store 10% off now, Then we buy 1 item price: $80.");
    // Call the price method on the CalculationContract with two arguments, 80 and 90, and log the return value.
    console.log("Subtract the discount, get the sale price:", CalculationContract.price(80, 90));
    // Log the explanation of the correct calculation.
    console.log("Multiply before dividing is correct. 80*90/100=72");
}
```
Red box: miscalculated price.
![Alt text](image-25.png)
