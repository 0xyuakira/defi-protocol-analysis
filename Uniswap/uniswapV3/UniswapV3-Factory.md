# Uniswap V3 Factory

> 和 V2 一样，`UniswapV3Factory`合约是所有 pool 合约的创建者，存储了所有 pool 合约的地址。在 V3 中，它的主要作用还包括费率档位管理和协议治理相关的功能。

## Storage

```solidity
    address public override owner;
    mapping(uint24 => int24) public override feeAmountTickSpacing;
    mapping(address => mapping(address => mapping(uint24 => address))) public override getPool;
```

1. owner

   - 它是 Factory 管理员的地址，初始为部署调用者，之后可以通过`setOwner`转移
   - 它的权限包括：启用新的费率档位，转移管理员权限，以及通过 Pool 的`setFeeProtocol`和`collectProtocol`设置和提取协议费

2. feeAmountTickSpacing

   - 和 V2 不同，在 V3 中一个交易对可以有多个手续费率，这里记录了不同手续费率 和 最小价格单位间隔 的映射，构造函数中的初始配置为：

     0.05% → tickSpacing = 10

     0.3% → tickSpacing = 60

     1% → tickSpacing = 200

3. getPool

   - 记录 (token0, token1, fee) → Pool 地址 的映射，确保同一交易对在一种手续费率下只有一个池子
   - 外部可以通过`getPool(tokenA, tokenB, fee)`快速找到对应的池子地址

## Constructor

```solidity
    constructor() {
        owner = msg.sender;
        emit OwnerChanged(address(0), msg.sender);

        feeAmountTickSpacing[500] = 10;
        emit FeeAmountEnabled(500, 10);
        feeAmountTickSpacing[3000] = 60;
        emit FeeAmountEnabled(3000, 60);
        feeAmountTickSpacing[10000] = 200;
        emit FeeAmountEnabled(10000, 200);
    }
```

- 初始化部署者为 owner
- 初始化 3 个常用手续费率：0.05%、0.3%、1%，对应 tickSpacing 分别为 10、60、200

## CreatePool

```solidity
function createPool(
    address tokenA,
    address tokenB,
    uint24 fee
) external override noDelegateCall returns (address pool) {
    // 1. 确保两个代币地址不同
    require(tokenA != tokenB, "identical tokens not allowed");

    // 2. 统一代币顺序：token0 永远是地址较小的代币
    //    这样 (WETH, USDC) 和 (USDC, WETH) 得到的 pool 是同一个
    (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);

    // 3. 检查 token0 地址是否合法（非零地址）
    require(token0 != address(0), "token0 is zero address");

    // 4. 查找 手续费率 档位对应的 tickSpacing
    //    例如：500 -> 10, 3000 -> 60, 10000 -> 200
    int24 tickSpacing = feeAmountTickSpacing[fee];
    require(tickSpacing != 0, "fee tier not enabled");

    // 5. 确保该 (token0, token1, fee) 的 pool 尚未被创建
    require(getPool[token0][token1][fee] == address(0), "pool already exists");

    // 6. 部署新的 pool 合约
    pool = deploy(address(this), token0, token1, fee, tickSpacing);

    // 7. 在映射中登记 pool 地址
    getPool[token0][token1][fee] = pool;
    // 同时存储反向映射，避免调用时还要比较顺序
    getPool[token1][token0][fee] = pool;

    // 8. 触发事件，供外部前端/SDK/indexer 监听
    emit PoolCreated(token0, token1, fee, tickSpacing, pool);
}
```

这里第 6 步，是调用了继承自`UniswapV3PoolDeployer`合约中的`deploy`函数。

```solidity
contract UniswapV3PoolDeployer is IUniswapV3PoolDeployer {
    // ========================
    // 临时存储结构体
    // ========================
    struct Parameters {
        address factory;
        address token0;
        address token1;
        uint24 fee;
        int24 tickSpacing;
    }

    Parameters public override parameters;


    function deploy(
        address factory,
        address token0,
        address token1,
        uint24 fee,
        int24 tickSpacing
    ) internal returns (address pool) {
        // 1. 把参数写入临时存储
        parameters = Parameters({
            factory: factory,
            token0: token0,
            token1: token1,
            fee: fee,
            tickSpacing: tickSpacing
        });

        // 2. 使用 CREATE2 部署新池子
        //    - salt: 确保相同 token0, token1, fee 的池子地址是唯一且可预测的
        pool = address(new UniswapV3Pool{
            salt: keccak256(abi.encode(token0, token1, fee))
        }());

        // 3. 部署完成后清理临时参数 parameters
        delete parameters;
    }
}
```

`UniswapV3Factory`继承了`UniswapV3PoolDeployer`合约，这里把参数写入了`UniswapV3Factory`的 storage 中，然后通过`create2`的方式部署了池子合约。那这些参数是怎么写入到池子合约的配置中呢？

```solidity
    constructor() {
        int24 _tickSpacing;
        (factory, token0, token1, fee, _tickSpacing) = IUniswapV3PoolDeployer(msg.sender).parameters();
        tickSpacing = _tickSpacing;

        maxLiquidityPerTick = Tick.tickSpacingToMaxLiquidityPerTick(_tickSpacing);
    }
```

上面是`UniswapV3Pool`合约的构造函数，可以看到它主动读取了部署者（也就是`UniswapV3Factory`）的 storage 变量`parameters`，然后再写入到自己的配置中。

完整流程是`createPool → initialize(sqrtPriceX96) → mint`。`initialize`设置初始价格和 Oracle 状态，只能成功调用一次，无需 owner 权限，也不会注入流动性。NPM 继承的`PoolInitializer.createAndInitializePoolIfNecessary`可以按需衔接创建与初始化。

## enableFeeAmount

```solidity
function enableFeeAmount(uint24 fee, int24 tickSpacing) public override {
    // 只有管理员(owner)可以启用新的手续费档位
    require(msg.sender == owner);

    // 手续费必须小于 100%（单位为百万分之一）
    require(fee < 1000000);

    // 限制 tick 间隔在合理范围 [1, 16383]
    // 避免 TickBitmap 计算溢出，同时保证价格精度
    require(tickSpacing > 0 && tickSpacing < 16384);

    // 确保该手续费档位还未被启用
    require(feeAmountTickSpacing[fee] == 0);

    // 保存手续费档位对应的 tick 间隔
    feeAmountTickSpacing[fee] = tickSpacing;

    // 触发事件，通知外部应用该手续费档位已启用
    emit FeeAmountEnabled(fee, tickSpacing);
}
```

这个函数由 owner 调用，用于增加新的手续费率和对应的 `tickSpacing`，不会修改现有 Pool 的固定费率和 tick 间隔。
