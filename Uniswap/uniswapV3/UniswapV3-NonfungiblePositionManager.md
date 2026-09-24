# Uniswap V3 NonfungiblePositionManager

## V3 中 LP 头寸

在 UniswapV2 中，同一池子的 LP 头寸都覆盖整个价格范围，因此可以用同质化的 ERC20 LP Token 表示份额。但是在 V3 中，每个 LP 的头寸都可能分布在不同的价格区间，不再同质化。

在`UniswapV3Pool`中，LP 的头寸用如下结构存储：

```solidity
mapping(bytes32 => Position.Info) public positions;
```

- key: 由`(owner, tickLower, tickUpper)`哈希得到
- value：存储了该头寸的流动性，单位流动性手续费累计量，待领取的本金和手续费等信息

这是最底层的状态存储，只为结算逻辑服务。Pool 不直接提供按用户枚举头寸的接口，也缺乏用户友好的凭证和入口，因此 V3 提供了`NonfungiblePositionManager`合约来解决这些问题，它有两个核心功能：

1. 把`position`封装成 NFT
   - 每个 LP 头寸对应一个 NFT
   - NFT 的`tokenId`唯一标识一个头寸
   - 用户可以存放或转移 NFT（转移头寸）
2. 提供更友好的 LP 操作入口
   - `mint`：调用`UniswapV3Pool`合约的`mint`创建头寸，并铸造一个 NFT
   - `increaseLiquidity`/ `decreaseLiquidity`: 调用 `UniswapV3Pool`的`mint`/ `burn`函数，调整头寸流动性
   - `collect`: 调用`UniswapV3Pool`合约的`collect`，提取头寸待领取的本金和手续费
   - `burn`： 销毁 NFT

**总结**：`UniswapV3Pool`中保存的是 LP 头寸的底层数据结构，只有底层结算逻辑。`NonfungiblePositionManager`把这些头寸抽象成 NFT，并暴露给用户更清晰更友好的接口。

## 源码解析

### Struct

```solidity
    struct Position {
        uint96 nonce;
        address operator;
        uint80 poolId;
        int24 tickLower;
        int24 tickUpper;
        uint128 liquidity;
        uint256 feeGrowthInside0LastX128;
        uint256 feeGrowthInside1LastX128;
        uint128 tokensOwed0; // 已记账但尚未领取的 token0（本金和手续费）
        uint128 tokensOwed1; // 已记账但尚未领取的 token1（本金和手续费）
    }
```

这里的`Position`就是 NFT 所记录的头寸数据，是`UniswapV3Pool`聚合头寸之上的 NFT 明细，并附带 NFT 管理逻辑。

- nonce：用于`ERC721Permit`场景，每个 NFT 都有一个 nonce，防止重放攻击
- operator：被授权管理该 NFT 的地址，可以通过`approve`或`permit`设置单个 NFT 的授权
- poolId：表示该 NFT 对应的头寸属于哪个池子的，因为同一交易对会有多个池子，这里没有存池子的地址，而是存一个索引，节省存储

### Storage

```solidity
    mapping(address => uint80) private _poolIds;
    mapping(uint80 => PoolAddress.PoolKey) private _poolIdToPoolKey;
    mapping(uint256 => Position) private _positions;
    uint176 private _nextId = 1;
    uint80 private _nextPoolId = 1;
    address private immutable _tokenDescriptor;
```

- \_poolIds：key 是池子合约地址，value 是分配给池子 id
- \_poolIdToPoolKey：key 是池子 id，value 是池子的{token0, token1, fee}
- \_positions：key 是 NFT 的唯一标识 tokenId，Position 就是上面提到的结构体，记录了头寸数据
  > 这三个`mapping`就把从头寸到池子完整串起来了： tokenId → position → poolId → poolKey → poolAddress
- \_nextId：下一个 NFT 的 id，每 mint 一个新的 Position 就自增
- \_nextPoolId：下一个要分配的 池子 id
- \_tokenDescriptor：指向一个用于生成 NFT 的`tokenURI`的合约地址，负责展示头寸的元数据，比如“WETH/USDC 0.3% fee, range 1200–1500”

### Constructor

```solidity
constructor(
        address _factory,
        address _WETH9,
        address _tokenDescriptor_
    ) ERC721Permit('Uniswap V3 Positions NFT-V1', 'UNI-V3-POS', '1') PeripheryImmutableState(_factory, _WETH9) {
        _tokenDescriptor = _tokenDescriptor_;
    }
```

1. ERC721Permit

   这是一个扩展过的 ERC721，实现了 EIP-2612 风格的 permit（允许 NFT 授权用签名完成，而不是链上的 approve）

   - `Uniswap V3 Positions NFT-V1` → NFT 名称
   - `UNI-V3-POS` → NFT 符号
   - `1` → 版本号（用于 EIP-712 域分隔符）

2. PeripheryImmutableState

   提供 `factory` 和 `WETH9` 两个 immutable 变量，方便调用，因为`NonfungiblePositionManager`合约经常需要知道池子合约地址是不是由官方 factory 合约部署的，以及处理 ETH 和 WETH 的转换。

### mint

```solidity
function mint(MintParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (
        uint256 tokenId,
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    )
{
    IUniswapV3Pool pool;

    // 1. 调用 addLiquidity：核心逻辑是
    //      a. 根据 factory 找到对应的 pool
    //      b. 调用 pool.mint() 增加流动性
    //      c. 在回调中把用户的 token 转给 pool
    (liquidity, amount0, amount1, pool) = addLiquidity(
        AddLiquidityParams({
            token0: params.token0,
            token1: params.token1,
            fee: params.fee,
            recipient: address(this),   // 注意：流动性挂在 NPM 合约自己名下
            tickLower: params.tickLower,
            tickUpper: params.tickUpper,
            amount0Desired: params.amount0Desired,
            amount1Desired: params.amount1Desired,
            amount0Min: params.amount0Min,
            amount1Min: params.amount1Min
        })
    );

    // 2. 铸造一张 NFT 给用户，NFT 的 ID 作为 position 的唯一标识
    _mint(params.recipient, (tokenId = _nextId++));

    // 3. 计算 positionKey
    // Pool 内部用 (owner, tickLower, tickUpper) 作为 position key
    // 这里 owner = NPM 合约地址
    bytes32 positionKey = PositionKey.compute(address(this), params.tickLower, params.tickUpper);

    // 4. 从 pool 拿到当前头寸的手续费累计值
    // feeGrowthInside0LastX128 / feeGrowthInside1LastX128 表示此时区间内的单位流动性手续费累计状态
    (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey);

    // 5. 绑定 poolId：把池子地址映射成一个整数 ID，方便 positions 存储
    uint80 poolId =
        cachePoolKey(
            address(pool),
            PoolAddress.PoolKey({token0: params.token0, token1: params.token1, fee: params.fee})
        );

    // 6. 把 position 数据写入 NPM合约 的存储映射
    _positions[tokenId] = Position({
        nonce: 0,                                // 用于 permit（签名授权）
        operator: address(0),                    // 可选代理地址
        poolId: poolId,                          // 池子 ID
        tickLower: params.tickLower,             // 区间下界
        tickUpper: params.tickUpper,             // 区间上界
        liquidity: liquidity,                    // 实际增加的流动性
        feeGrowthInside0LastX128: feeGrowthInside0LastX128, // 记录当时的手续费状态
        feeGrowthInside1LastX128: feeGrowthInside1LastX128,
        tokensOwed0: 0,                          // 初始时没有待领取的本金和手续费
        tokensOwed1: 0
    });

    // 7. 事件：方便前端和链上工具追踪
    emit IncreaseLiquidity(tokenId, liquidity, amount0, amount1);
}
```

我们可以看到`mint`中一个很重要的点是：在调用 pool 合约的`mint`方法时，流动性的 owner 是 NPM 合约地址，NPM 合约中记录了 position 的流动性信息，这也方便了后续通过 NPM 合约进行其他管理流动性的操作。

### increaseLiquidity

```solidity
function increaseLiquidity(IncreaseLiquidityParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (
        uint128 liquidity,
        uint256 amount0,
        uint256 amount1
    )
{
    // 1. 找到这个 NFT 对应的 position（通过 tokenId）
    Position storage position = _positions[params.tokenId];

    // 2. 根据 poolId 找到对应池子的 key（token0, token1, fee）
    PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId];

    IUniswapV3Pool pool;
    // 3. 调用 addLiquidity → 调用 pool.mint()
    (liquidity, amount0, amount1, pool) = addLiquidity(
        AddLiquidityParams({
            token0: poolKey.token0,
            token1: poolKey.token1,
            fee: poolKey.fee,
            tickLower: position.tickLower,
            tickUpper: position.tickUpper,
            amount0Desired: params.amount0Desired,
            amount1Desired: params.amount1Desired,
            amount0Min: params.amount0Min,
            amount1Min: params.amount1Min,
            recipient: address(this)             // 仍然挂在 NPM 合约上
        })
    );

    // 4. 计算 positionKey（pool 内部索引用）
    bytes32 positionKey = PositionKey.compute(address(this), position.tickLower, position.tickUpper);

    // 5. 从 pool 拿到最新的 feeGrowthInside
    (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey);

    // 6. 计算之前积累的手续费并加到 tokensOwed
    position.tokensOwed0 += uint128(
        FullMath.mulDiv(
            feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128, // 手续费增量
            position.liquidity,                                          // 乘以旧流动性
            FixedPoint128.Q128                                           // 还原精度
        )
    );
    position.tokensOwed1 += uint128(
        FullMath.mulDiv(
            feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
            position.liquidity,
            FixedPoint128.Q128
        )
    );

    // 7. 更新最新的 feeGrowthInside
    position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128;
    position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128;

    // 8. 最终增加流动性（旧的 + 新的）
    position.liquidity += liquidity;

    // 9. 事件
    emit IncreaseLiquidity(params.tokenId, liquidity, amount0, amount1);
}
```

这里需要注意第 6 步，因为 NPM 合约托管了一大堆 NFT，pool 合约里看到的只会是(owner = NPM, tickLower, tickUpper)，所以这里需要 NPM 合约自己细分每个 NFT 的欠款。也就是说 pool 合约中`tokensOwed`是聚合账本，按 (owner 地址, tick 区间) 记的，NPM 的 tokensOwed 是“细账”，按 NFT id 记的。

### decreaseLiquidity

`increaseLiquidity`允许调用者为现有 NFT 支付资产并增加流动性，不要求持有或获授权操作该 NFT；`decreaseLiquidity`则要求 NFT owner 或授权者调用，检查 deadline，并要求撤出的 liquidity 大于 0 且不超过该头寸的流动性。

它调用 pool 合约的`burn`，用 amount0Min/amount1Min 检查本次撤出的本金，将本金和新增手续费记入该 NFT 的 tokensOwed，之后通过`collect`转账。

### collect

```solidity
function collect(CollectParams calldata params)
    external
    payable
    override
    isAuthorizedForToken(params.tokenId)   // NFT owner 或 approved operator 调用
    returns (uint256 amount0, uint256 amount1)
{
    require(params.amount0Max > 0 || params.amount1Max > 0);

    // 2. 如果传入的 recipient 是 0 地址，就默认收款地址为自己（NPM合约）
    address recipient = params.recipient == address(0) ? address(this) : params.recipient;

    // 3. 根据 tokenId 找到 Position 存储
    Position storage position = _positions[params.tokenId];

    // 4. 通过 poolId 找到对应的 poolKey，进而拿到 Pool 地址
    PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId];
    IUniswapV3Pool pool = IUniswapV3Pool(PoolAddress.computeAddress(factory, poolKey));

    // 5. 拿到当前记录的 tokensOwed（NPM 自己维护的账本）
    (uint128 tokensOwed0, uint128 tokensOwed1) = (position.tokensOwed0, position.tokensOwed1);

    // 6. 如果头寸还有流动性，就要更新手续费快照
    if (position.liquidity > 0) {
        // 调用 pool.burn(..., liquidity=0)，结算 Pool 头寸的手续费并更新快照，不减少流动性
        pool.burn(position.tickLower, position.tickUpper, 0);

        // 从 pool 中读取最新的 feeGrowthInside
        (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) =
            pool.positions(PositionKey.compute(address(this), position.tickLower, position.tickUpper));

        // 算出新增的手续费欠款
        tokensOwed0 += uint128(
            FullMath.mulDiv(
                feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
                position.liquidity,
                FixedPoint128.Q128
            )
        );
        tokensOwed1 += uint128(
            FullMath.mulDiv(
                feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
                position.liquidity,
                FixedPoint128.Q128
            )
        );

        // 更新 NPM 内部的快照
        position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128;
        position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128;
    }

    // 7. 确定本次最多提取多少（受 amount0Max / amount1Max 限制）
    (uint128 amount0Collect, uint128 amount1Collect) = (
        params.amount0Max > tokensOwed0 ? tokensOwed0 : params.amount0Max,
        params.amount1Max > tokensOwed1 ? tokensOwed1 : params.amount1Max
    );

    // 8. 调用 pool.collect，把代币真正转给 recipient
    (amount0, amount1) = pool.collect(
        recipient,
        position.tickLower,
        position.tickUpper,
        amount0Collect,
        amount1Collect
    );

    // 9. 更新 position.tokensOwed，减去本次领取的部分
    (position.tokensOwed0, position.tokensOwed1) = (tokensOwed0 - amount0Collect, tokensOwed1 - amount1Collect);

    emit Collect(params.tokenId, recipient, amount0Collect, amount1Collect);
}
```

### burn

```solidity
function burn(uint256 tokenId) external payable override isAuthorizedForToken(tokenId) {
    Position storage position = _positions[tokenId];
    // 1. 确保流动性和待领取的本金、手续费都清零
    require(position.liquidity == 0 && position.tokensOwed0 == 0 && position.tokensOwed1 == 0, 'Not cleared');

    // 2. 删除 position 映射里的存储
    delete _positions[tokenId];

    // 3. 销毁对应的 ERC721 token
    _burn(tokenId);
}
```

这里的`burn`是销毁 NFT 方法，不会调用 pool 合约的撤销流动性方法，如果这个 NFT 没有流动性，也没有待领取的本金和手续费，就可以销毁。一般流程是：

- 调用 decreaseLiquidity（内部会调 pool.burn），把头寸 liquidity 取出
- 调用 collect，把欠款 tokens 取走
- 最后调用 NonfungiblePositionManager.burn(tokenId)，销毁这个 NFT
