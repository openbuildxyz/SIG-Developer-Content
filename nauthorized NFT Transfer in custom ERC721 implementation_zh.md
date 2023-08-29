[NFT-transfer.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/NFT-transfer.sol)

# 定制 ERC721 里未经授权的 NFT转账

**名称：** 自定义 ERC721 实施中未经授权的 NFT 传输。

**描述：** 合约ERC721 中的定制 TransferFrom 函数无法正确检查 msg.sender 是否是代币的当前所有者地址或已审核的地址。 因此，任何地址都可以调用transferFrom函数来转移任何代币，无论当前所有者是谁。 这使得未经授权的用户可以转账不属于他们的代币，从而导致资产被盗。

**解决办法：**

确保 msg.sender 是令牌的当前所有者或已审核的地址。

**参考：**

https://twitter.com/1nf0s3cpt/status/1679120390281412609

https://blog.decurity.io/scanning-for-vulnerable-erc721-implementations-fe19200b91b5

https://ventral.digital/posts/2022/8/18/sznsdaos-bountyboard-unauthorized-transferfrom-vulnerability

https://github.com/pessimistic-io/slitherin/blob/master/docs/nft_approve_warning.md

**VulnerableERC721漏洞合约:**

```jsx
contract VulnerableERC721 is ERC721, Ownable {
    constructor() ERC721("MyNFT", "MNFT") {}

    //自定义的transferFrom函数缺少NFT所有者检查。
    function transferFrom(
        address from,
        address to,
        uint256 tokenId
    ) public override {
        // 直接转账
        _transfer(from, to, tokenId);
    }

    function safeMint(address to, uint256 tokenId) public onlyOwner {
        _safeMint(to, tokenId);
    }
}
```

**ERC721 已修复合约:(紫色字体)**

```jsx
contract FixedERC721 is ERC721, Ownable {
    constructor() ERC721("MyNFT", "MNFT") {}

    //措施：添加令牌所有者检查
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

***\*测试方法:\****

**forge test --contracts src/test/**NFT-transfer.sol**-vvvv**

```jsx
// 测试函数以演示 VulnerableERC721Contract 中的漏洞。
function testVulnerableERC721() public {
    // 调用VulnerableERC721Contract的'ownerOf'函数来检查代币ID 1的当前所有者。
    VulnerableERC721Contract.ownerOf(1);

    // 调用虚拟机 (vm) 的“prank”函数并传递“bob”作为参数。
     // 目前还不清楚“恶作剧”函数到底做什么，因为它取决于虚拟机的实现。
     // 意图很可能是模拟一些与“bob”相关的行为。
    vm.prank(bob);

    // 调用 VulnerableERC721Contract 的“transferFrom”函数。
     // 尝试将令牌 ID 1 从 'alice' 传输到 'bob'。
    VulnerableERC721Contract.transferFrom(address(alice), address(bob), 1);

    //尝试转账后再次调用 'ownerOf' 函数来检查代币 ID 1 的所有者。
     // 目标是观察转移是否按预期发生或者漏洞是否影响所有权。
    console.log(VulnerableERC721Contract.ownerOf(1));
}
```

**红框：**合同ERC721中的定制transferFrom函数未正确检查msg.sender是否为令牌的当前所有者或已审核的地址**紫色盒子：**问题已修复。

![img](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F5385296b-1aa2-4213-9a28-ce04188bffd1%2FUntitled.png?table=block&id=d29523b8-08d8-41c8-9229-72ecb1efc56d&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)