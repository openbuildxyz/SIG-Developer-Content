# Unauthorized NFT Transfer in custom ERC721 implementation

**Name:** Unauthorized NFT Transfer in custom ERC721 implementation.

**Description:** Custom transferFrom function in contract VulnerableERC721, does not properly check if msg.sender is the current owner of the token or an approved address. As a result, any address can call the transferFrom function to transfer any token, regardless of who the current owner is. This allows unauthorized users to transfer tokens they do not own, leading to potential theft of assets.

**Mitigation:**

To ensure that msg.sender is the current owner of the token or an approved address.

**REF:**

https://twitter.com/1nf0s3cpt/status/1679120390281412609

https://blog.decurity.io/scanning-for-vulnerable-erc721-implementations-fe19200b91b5

https://ventral.digital/posts/2022/8/18/sznsdaos-bountyboard-unauthorized-transferfrom-vulnerability

https://github.com/pessimistic-io/slitherin/blob/master/docs/nft_approve_warning.md

**VulnerableERC721 Contract:**

```jsx
contract VulnerableERC721 is ERC721, Ownable {
    constructor() ERC721("MyNFT", "MNFT") {}

    //custom transferFrom function which missing NFT owner check.
    function transferFrom(
        address from,
        address to,
        uint256 tokenId
    ) public override {
        // direct transfer
        _transfer(from, to, tokenId);
    }

    function safeMint(address to, uint256 tokenId) public onlyOwner {
        _safeMint(to, tokenId);
    }
}
```

**FixedERC721 Contract:(**Purple Words)

```jsx
contract FixedERC721 is ERC721, Ownable {
    constructor() ERC721("MyNFT", "MNFT") {}

    //Mitigation: add token owner check
    function transferFrom(
        address from,
        address to,
        uint256 tokenId
    ) public override {
        **require(
            _isApprovedOrOwner(_msgSender(), tokenId),**
            "ERC721: caller is not token owner or approved"
        );

        _transfer(from, to, tokenId);
    }

    function safeMint(address to, uint256 tokenId) public onlyOwner {
        _safeMint(to, tokenId);
    }
```

***\*How to Test:\****

forge test --contracts src/test/**NFT-transfer.sol**-vvvv

```jsx
// Test function to demonstrate a vulnerability in the VulnerableERC721Contract.
function testVulnerableERC721() public {
    // Call the 'ownerOf' function of the VulnerableERC721Contract to check the current owner of token ID 1.
    VulnerableERC721Contract.ownerOf(1);

    // Call the 'prank' function of the virtual machine (vm) and pass 'bob' as an argument.
    // It's unclear what exactly the 'prank' function does, as it depends on the implementation of the virtual machine.
    // The intention is likely to simulate some behavior related to 'bob'.
    vm.prank(bob);

    // Call the 'transferFrom' function of the VulnerableERC721Contract.
    // Attempt to transfer token ID 1 from 'alice' to 'bob'.
    VulnerableERC721Contract.transferFrom(address(alice), address(bob), 1);

    // Call the 'ownerOf' function again to check the owner of token ID 1 after the transfer attempt.
    // The goal is to observe if the transfer has occurred as expected or if the vulnerability affects the ownership.
    console.log(VulnerableERC721Contract.ownerOf(1));
}
```

**Red box:** Custom transferFrom function in contract VulnerableERC721, does not properly check if msg.sender is the current owner of the token or an approved address. **Purple box:** issue fixed.

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F5385296b-1aa2-4213-9a28-ce04188bffd1%2FUntitled.png?table=block&id=d29523b8-08d8-41c8-9229-72ecb1efc56d&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)