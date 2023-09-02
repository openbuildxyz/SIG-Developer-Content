# 脏字节 
[Dirtybytes.sol](https://github.com/SunWeb3Sec/DeFiVulnLabs/blob/main/src/test/Dirtybytes.sol)  

**名称：** Solidity 0.8.15 及以上版本的脏字节问题  

**描述：**   


在将memory或calldata中的bytes数组复制到storage中时，即使长度不是32的倍数，也会以32字节的块来进行复制。  
因此，超过数组末尾的额外字节可能会从calldata或memory复制到storage中。  
这些脏字节可能在对存储中的bytes数组执行无参数的.push()操作后变得可观察，即这样的推送不会如预期那样在数组末尾产生零值。  
此错误仅影响旧的代码生成流程，通过IR的新代码生成流程不受影响。。  
```
"link": <https://blog.soliditylang.org/2022/06/15/dirty-bytes-array-to-storage-bug/>
"fixed": 0.8.15
``` 
**合约：**  
```
contract ContractTest is Test {
    Dirtybytes Dirtybytesontract;

    function testDirtybytes() public {
        Dirtybytesontract = new Dirtybytes();
        emit log_named_bytes(
            "Array element in h() not being zero::",
            Dirtybytesontract.h()
        );
        console.log(
            "Such that the byte after the 63 bytes allocated below will be 0x02."
        );
    }
}

contract Dirtybytes {
    event ev(uint[], uint);
    bytes s;

    constructor() {
        // 下面的事件发射涉及在当前空闲内存指针的位置上写入临时内存。
        //  其他几个操作（例如某些keccak256调用）将以类似的方式使用临时内存。
        // 在这种特殊情况下，传递的数组的长度将被精确地写入临时内存
        // 以使得在下面分配的63字节后的字节为0x02  
        // 这个脏字节随后将在赋值过程中写入存储，并在h中的推送操作中变得可见。
        emit ev(new uint[](2), 0);
        bytes memory m = new bytes(63);
        s = m;
    }

    function h() external returns (bytes memory) {
        s.push();
        return s;
    }
}
```  
**如何测试:**   

forge test --contracts src/test/Dirtybytes.sol -vvvv  

**红色框：** 在 h() 中的数组元素未被置为零。  

![image](https://web3sec.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Ff2080506-6b0b-41c3-802f-795617f0556a%2FUntitled.png?table=block&id=847240ad-edfb-413b-8b8f-9d9e66221f70&spaceId=369b5001-5511-4fe6-a099-48af1d841f20&width=2000&userId=&cache=v2)
