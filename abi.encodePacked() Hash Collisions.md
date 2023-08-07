# abi.encodePacked() Hash Collisions

**Name:** abi.encodePacked() Hash Collisions

**Description:** Using abi.encodePacked() with multiple variable length arguments can, in certain situations, lead to a hash collision.

Hash functions are designed to be unique for each input, but collisions can still occur due to limitations in the hash function's size or the sheer number of possible inputs. This is a known issue mentioned:https://docs.soliditylang.org/en/v0.8.17/abi-spec.html?highlight=collisions#non-standard-packed-mode This is a known issue mentioned: https://docs.soliditylang.org/en/v0.8.17/abi-spec.html?highlight=collisions#non-standard-packed-mode

In deposit function allows users to deposit Ether into the contract based on two string inputs: _string1 and _string2. The contract uses the keccak256 function to generate a unique hash by concatenating these two strings.

If two different combinations of _string1 and _string2 produce the same hash value, a hash collision will occur. The code does not handle this scenario properly and allows the second depositor to overwrite the previous deposit.

**Mitigation:** use of abi.encode() instead of abi.encodePacked()

**REF:**

https://twitter.com/1nf0s3cpt/status/1676476475191750656

https://docs.soliditylang.org/en/v0.8.17/abi-spec.html?highlight=collisions#non-standard-packed-mode

https://swcregistry.io/docs/SWC-133

https://github.com/sherlock-audit/2022-10-nftport-judging/issues/118

**HashCollisionBug Contract:**

```jsx
contract HashCollisionBug {
    mapping(bytes32 => uint256) public balances;

    function createHash(
        string memory _string1,
        string memory _string2
    ) public pure returns (bytes32) {
        return keccak256(abi.encodePacked(_string1, _string2));
    }

    function deposit(
        string memory _string1,
        string memory _string2
    ) external payable {
        require(msg.value > 0, "Deposit amount must be greater than zero");

        bytes32 hash = createHash(_string1, _string2);
        // createHash(AAA, BBB) -> AAABBB
        // createHash(AA, ABBB) -> AAABBB
        // Check if the hash already exists in the balances mapping
        require(balances[hash] == 0, "Hash collision detected");

        balances[hash] = msg.value;
    }
}
```

***\*How to Test:\****

forge test --contracts src/test/**Hash-collisions.sol**-vvvv

```jsx
// Test function to check for hash collision bug in the HashCollisionBugContract.
function testHash_collisions() public {
    // Emit a log with the name and the result of the hash calculation for inputs "AAA" and "BBB".
    emit log_named_bytes32(
        "(AAA,BBB) Hash",
        HashCollisionBugContract.createHash("AAA", "BBB")
    );

    // Call the 'deposit' function of the HashCollisionBugContract with value 1 ether and inputs "AAA" and "BBB".
    HashCollisionBugContract.deposit{value: 1 ether}("AAA", "BBB");

    // Emit a log with the name and the result of the hash calculation for inputs "AA" and "ABBB".
    emit log_named_bytes32(
        "(AA,ABBB) Hash",
        HashCollisionBugContract.createHash("AA", "ABBB")
    );

    // Expect a revert with the message "Hash collision detected" for the following transaction.
    // Call the 'deposit' function of the HashCollisionBugContract with value 1 ether and inputs "AA" and "ABBB".
    // This transaction is expected to revert because it might trigger a hash collision bug.
    vm.expectRevert("Hash collision detected");
    HashCollisionBugContract.deposit{value: 1 ether}("AA", "ABBB"); //Hash collision detected
}
```

**Red box: same hash.**

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fe334209d-2420-4a6d-a363-22cbb2d81756%2FUntitled.png?table=block&id=6e6f8553-0929-4f95-ad76-51f2f50ef00f&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)