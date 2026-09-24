# Uniswap V3 Mint & Burn

## uniswapV2 和 V3 的区别

**V2: 全区间，统一池，被动参与**

- LP 添加的流动性覆盖整个价格区间 $(0,\infty)$。无论当前市场价格处于哪个位置，LP 的资金总是会参与报价。
- 所有资金都放在一个共享的大池子中，添加新的流动性只需要按照当前市场价格比例，等值提供两种 token。

因此，V2 中的流动性可以直接根据 LP 添加的 token 数量来计算。

**V3: 指定价格区间，多池，主动控制**

- LP 不再被迫提供全区间流动性，可以指定在特定价格区间提供流动性，每个价格区间都有自己的价格曲线，流动性计算也是独立的。只有当市场价格在该价格区间运行时，LP 的资金才会参与报价。
- 当在非活跃价格区间提供流动性，只需提供其中一种 token，也就是可以提供单边流动性。

因此，V3 中头寸流动性 L 由 token 数量、LP 指定的价格区间和当前市场价格共同决定。

## 数学模型

<img src="images/uniswapV3-03.png" alt="uniswapV3 计算流动性" width="50%" height="50%">

假设存在交易对 (x,y)，LP 设定添加流动性的区间为 $[P_{lower},P_{upper}]$ , 当前市场价格为 $P$ , 在 $P$ 点头寸对应的资产数量为 $\Delta x$ , $\Delta y$ 。然后我们分析以下三种情况：

1. **$P_{lower}<P<P_{upper}$**

   当前市场价格位于这个价格区间内时，我们可以发现从 $P$ 到 $P_{lower}$ 这段价格曲线的流动性实际上是由资产 y 支撑的，因为价格从 $P$ 到 $P_{lower}$ 消耗的是池子中资产 y。 $P$ 到 $P_{upper}$ 的流动性是由资产 x 支撑的，价格从 $P$ 到 $P_{upper}$ 消耗的是池子中资产 x。然后计算 $\Delta x$ 和 $\Delta y$ ：

$$
\Delta x = x_p - x_{upper} = \frac{L}{\sqrt{P}} - \frac{L}{\sqrt{P_{upper}}} = L(\frac{1}{\sqrt{P}} - \frac{1}{\sqrt{P_{upper}}})
$$

$$
\Delta y = y_p - y_{lower} = L \cdot \sqrt{P} - L \cdot \sqrt{P_{lower}} = L(\sqrt{P} - \sqrt{P_{lower}})
$$

根据上面的公式，可以反推出 L：

$$
L = \frac{\Delta x \cdot \sqrt{P} \cdot \sqrt{P_{upper}}}{\sqrt{P_{upper}} - \sqrt{P}}
$$

$$
L = \frac{\Delta y}{\sqrt{P} - \sqrt{P_{lower}}}
$$

2. **$P>= P_{upper}$**

   当前市场价格大于等于这个价格区间的上边界时，只需提供资产 y。从 $P_{upper}$ 到 $P_{lower}$ 这段价格曲线的流动性是由资产 y 支撑的，计算价格从 $P_{upper}$ 下跌到 $P_{lower}$ 消耗池子中资产 y 的数量 $\Delta y$ ：

$$
 \Delta y = y_{upper} - y_{lower} = L \cdot \sqrt{P_{upper}} - L \cdot \sqrt{P_{lower}} = L(\sqrt{P_{upper}} - \sqrt{P_{lower}})
$$

反推 L：

$$
L = \frac{\Delta y}{\sqrt{P_{upper}} - \sqrt{P_{lower}}}
$$

3. **$P<= P_{lower}$**

   当前市场价格小于等于这个价格区间的下边界时，同理，只需提供资产 x。从 $P_{lower}$ 到 $P_{upper}$ 这段价格曲线的流动性是由资产 x 支撑的，计算价格从 $P_{lower}$ 上涨到 $P_{upper}$ 消耗池子中资产 x 的数量 $\Delta x$ :

$$
\Delta x = x_{lower} - x_{upper} = \frac{L}{\sqrt{P_{lower}}} - \frac{L}{\sqrt{P_{upper}}} = L(\frac{1}{\sqrt{P_{lower}}} - \frac{1}{\sqrt{P_{upper}}})
$$

反推 L：

$$
L = \frac{\Delta x \cdot \sqrt{P_{lower}} \cdot \sqrt{P_{upper}}}{\sqrt{P_{upper}} - \sqrt{P_{lower}}}
$$

利用上面的公式计算实际添加的流动性时，还需注意：

- 先看第一种情况，当市场价格严格位于区间内部时，我们可以看到添加两种 token 会计算出两个 L，那么实际上 uniswapV3 要使用哪个呢？答案是小的那个，因为 L 必须同时被两种资产支持，取大的那个会导致其中一种资产不足。再按该 L 计算实际需要支付的两种 token 数量，NPM 在 mint 回调中只支付这些数量，未使用的 ERC20 无需转入；若使用原生 ETH 支付，剩余 ETH 需要通过 `refundETH` 取回。
- 第二和第三种情况本质上是一样的，LP 只需要添加其中一种 token，也就是所谓的单边流动性。价格严格位于区间外时，流动性暂不参与报价；精确边界处则按 `tickLower <= slot0.tick < tickUpper` 判断是否活跃，不能仅根据单边资产判断。

## 源码实现

### Position

在 tick 章节，我们学习了 uniswapV3 是如何记录和计算各个区间的流动性和手续费的。那么系统是如何记录 LP 的头寸和手续费收益呢？下面看`position`这个库是如何实现的。

#### Position.Info

```solidity
struct Info {
    uint128 liquidity; // 该 position 当前的流动性
    uint256 feeGrowthInside0LastX128;
    uint256 feeGrowthInside1LastX128;
    uint128 tokensOwed0; // 已记账但尚未领取的 token0（本金和手续费）
    uint128 tokensOwed1; // 已记账但尚未领取的 token1（本金和手续费）
}
```

- feeGrowthInside0LastX128/feeGrowthInside1LastX128

  这两个字段记录上次更新此 position 时，所在区间每单位流动性的累计手续费快照。所以如果要计算两次操作间 LP 应得多少手续费，只需要：
  $(feeGrowthInsideNow - feeGrowthInsideLast) * liquidity$

  这里使用本次变更前的 liquidity；如果直接代入 X128 存储值，还需除以 $2^{128}$ 并向下取整。

- tokensOwed0/tokensOwed1

  已经结算但 LP 尚未领取的手续费，以及 burn 后待领取的本金，都累加在这里。

手续费单独记账，不会自动增加头寸的 L，领取或再投入需要后续操作。协议费若开启，从收取的 token 手续费中分出，不通过增发 LP Token 收取。

#### get

```solidity
function get(
    mapping(bytes32 => Info) storage self,
    address owner,
    int24 tickLower,
    int24 tickUpper
) internal view returns (Position.Info storage position) {
    position = self[keccak256(abi.encodePacked(owner, tickLower, tickUpper))];
}
```

- 参数`mapping(bytes32 => Info) storage self` 是存放了所有 position 的映射表
- 每个 position 都可以通过 LP 地址（owner），tickLower，tickUpper 生成的唯一标识来确定，也就是对这三个信息编码后去哈希确定在 self 中 key：`keccak256(abi.encodePacked(owner, tickLower, tickUpper))`

#### update

这个函数是`Position`的核心逻辑，它主要处理了一个 LP 头寸在添加或移除流动性或累积手续费时，内部状态是如何更新的。

```solidity
function update(
    Info storage self,
    int128 liquidityDelta, //流动性变化量
    uint256 feeGrowthInside0X128, //当前tick区间的手续费累积值
    uint256 feeGrowthInside1X128
) internal {
    // 拷贝一份 position 的旧状态（避免直接操作 storage）
    Info memory _self = self;

    uint128 liquidityNext;
    if (liquidityDelta == 0) {
        // 情况一：只更新手续费，不改变流动性（poke 操作）
        // 但是必须保证这个头寸存在（liquidity > 0），否则报错 "NP"
        require(_self.liquidity > 0, 'NP');
        liquidityNext = _self.liquidity;
    } else {
        // 情况二：LP 增加或减少流动性
        // 调用 LiquidityMath.addDelta 来更新流动性（自动处理正负数）
        liquidityNext = LiquidityMath.addDelta(_self.liquidity, liquidityDelta);
    }

    // ================================
    // 计算自上次更新以来新增长的手续费
    // ================================

    // feeGrowthInside0X128 - _self.feeGrowthInside0LastX128
    // → 表示 LP 这个区间在 token0 上新增长的 fee（per liquidity）
    // 乘以 LP 的 liquidity → 得到 LP 新增的 token0 数量
    // 再除以 Q128（因为是 Q128.128 定点数）→ 转为整数
    uint128 tokensOwed0 =
        uint128(
            FullMath.mulDiv(
                feeGrowthInside0X128 - _self.feeGrowthInside0LastX128,
                _self.liquidity,
                FixedPoint128.Q128
            )
        );

    // 同理计算 token1 的新增手续费
    uint128 tokensOwed1 =
        uint128(
            FullMath.mulDiv(
                feeGrowthInside1X128 - _self.feeGrowthInside1LastX128,
                _self.liquidity,
                FixedPoint128.Q128
            )
        );

    // ================================
    // 更新 position 的状态
    // ================================

    // 如果 liquidityDelta ≠ 0，说明有增减流动性 → 写回 storage
    if (liquidityDelta != 0) self.liquidity = liquidityNext;

    // 更新 feeGrowthInside 的快照
    // 这样下次再调用 update 时，就能算新增手续费
    self.feeGrowthInside0LastX128 = feeGrowthInside0X128;
    self.feeGrowthInside1LastX128 = feeGrowthInside1X128;

    // 如果算出的 tokensOwed 大于 0，就累加到 position 上
    // 注意：这里只是记账，不会立即转账
    if (tokensOwed0 > 0 || tokensOwed1 > 0) {
        // 允许溢出（只要 LP 在 type(uint128).max 前取出即可）
        self.tokensOwed0 += tokensOwed0;
        self.tokensOwed1 += tokensOwed1;
    }
}
```

总结，`update`主要干了三件事：

1. 更新流动性
   - 如果有`liquidityDelta`，改`liquidity`
   - 如果只是 poke，维持原状
2. 更新手续费
   - 按前面的公式，用区间手续费增量和变更前的 liquidity 计算新增手续费
   - 把这部分手续费累加到`tokensOwed0` / `tokensOwed1`
3. 更新快照
   - 把`feeGrowthInsideLastX128`更新为该区间最新的单位流动性手续费累计值，确保下次计算手续费正确

所以本质上，`update`是`position`的记账逻辑，将该头寸的流动性和手续费结算到本次调用时。swap 不会逐个刷新所有 position 的快照和 owed，新增手续费在后续 mint、burn（包括 burn(0)）时结算。

### UniswapV3Pool.\_modifyPosition

在学习`mint`,`burn`之前，我们先学习，`UniswapV3Pool._modifyPosition`这个内部函数，它是 uniswapV3 处理添加移除流动性的核心部分，也被`mint`和`burn`调用。

```solidity
function _modifyPosition(ModifyPositionParams memory params)
    private
    noDelegateCall
    returns (
        Position.Info storage position,
        int256 amount0,
        int256 amount1
    )
{
    // 1. 校验 tickLower < tickUpper，且不超出 MIN_TICK / MAX_TICK
    // tickSpacing 对齐由后续 TickBitmap.flipTick 检查
    checkTicks(params.tickLower, params.tickUpper);

    // 读取 slot0 缓存，减少 SLOAD
    Slot0 memory _slot0 = slot0;

    // 2. 更新 position 和 tick 信息
    position = _updatePosition(
        params.owner,
        params.tickLower,
        params.tickUpper,
        params.liquidityDelta,
        _slot0.tick
    );

    if (params.liquidityDelta != 0) {
        // 3. 根据当前价格与 tick 区间的关系分三种情况计算 token 数量

        if (_slot0.tick < params.tickLower) {
            // 当前价格在区间下方：只需计算 token0
            amount0 = SqrtPriceMath.getAmount0Delta(
                TickMath.getSqrtRatioAtTick(params.tickLower),
                TickMath.getSqrtRatioAtTick(params.tickUpper),
                params.liquidityDelta
            );

        } else if (_slot0.tick < params.tickUpper) {
            // 当前价格在区间内：计算 token0 和 token1
            uint128 liquidityBefore = liquidity; // 缓存池子当前的活跃流动性

            // 4. 在改变活跃 liquidity 前写入 Oracle 观察值
            (slot0.observationIndex, slot0.observationCardinality) = observations.write(
                _slot0.observationIndex,
                _blockTimestamp(),
                _slot0.tick,
                liquidityBefore,
                _slot0.observationCardinality,
                _slot0.observationCardinalityNext
            );

            // 计算 token 数量
            amount0 = SqrtPriceMath.getAmount0Delta(
                _slot0.sqrtPriceX96,
                TickMath.getSqrtRatioAtTick(params.tickUpper),
                params.liquidityDelta
            );
            amount1 = SqrtPriceMath.getAmount1Delta(
                TickMath.getSqrtRatioAtTick(params.tickLower),
                _slot0.sqrtPriceX96,
                params.liquidityDelta
            );

            // 5. 更新池子当前的活跃 liquidity
            liquidity = LiquidityMath.addDelta(liquidityBefore, params.liquidityDelta);

        } else {
            // 当前价格在区间上方：只需计算 token1
            amount1 = SqrtPriceMath.getAmount1Delta(
                TickMath.getSqrtRatioAtTick(params.tickLower),
                TickMath.getSqrtRatioAtTick(params.tickUpper),
                params.liquidityDelta
            );
        }
    }
}
```

其中 `observations.write` 的观察值写入见 [Oracle 章节](./UniswapV3-Oracle.md)。

#### 1.Slot0

`Slot0`是`UniswapV3Pool`中定义的一个结构体，`slot0`将池子的常用状态打包保存在第一个 storage slot 中。我们这里主要用到了`sqrtPriceX96`,`tick`。

```solidity
struct Slot0 {
    uint160 sqrtPriceX96;           // 当前价格的平方根，Q64.96 定点数
    int24 tick;                     // 当前 tick，精确边界处与穿越方向有关
    uint16 observationIndex;        // 最新一次观察值在 observations 数组中的索引
    uint16 observationCardinality;  // 当前观察值环形缓冲区容量
    uint16 observationCardinalityNext; // 观察值容量的扩容目标
    uint8 feeProtocol;              // token0 / token1 的协议费分母，分别存于低 / 高四位
    bool unlocked;                  // 池子是否解锁，防止 reentrancy
}
```

#### 2.\_updatePosition

`_updatePosition`的逻辑主要是更新上下边界 tick 的状态，LP 的头寸 position 的状态。

```solidity
function _updatePosition(
    address owner,
    int24 tickLower,
    int24 tickUpper,
    int128 liquidityDelta,
    int24 tick
) private returns (Position.Info storage position) {
    // 1️⃣ 获取 LP 在 tickLower 和 tickUpper 区间的 position
    position = positions.get(owner, tickLower, tickUpper);

    uint256 _feeGrowthGlobal0X128 = feeGrowthGlobal0X128; // 全局每单位流动性的 token0 累计手续费
    uint256 _feeGrowthGlobal1X128 = feeGrowthGlobal1X128; // 全局每单位流动性的 token1 累计手续费

    bool flippedLower;
    bool flippedUpper;

    if (liquidityDelta != 0) {
        uint32 time = _blockTimestamp();

        // 2️⃣ 读取或补算当前时刻的 Oracle 累计值
        (int56 tickCumulative, uint160 secondsPerLiquidityCumulativeX128) =
            observations.observeSingle(
                time,
                0,
                slot0.tick,
                slot0.observationIndex,
                liquidity,
                slot0.observationCardinality
            );

        // 3️⃣ 更新下边界 tick 的状态
        flippedLower = ticks.update(
            tickLower,              // tick 下边界
            tick,                   // 当前价格所在 tick
            liquidityDelta,         // 增加或减少的流动性
            _feeGrowthGlobal0X128,  // 全局每单位流动性的 token0 累计手续费
            _feeGrowthGlobal1X128,  // 全局每单位流动性的 token1 累计手续费
            secondsPerLiquidityCumulativeX128,
            tickCumulative,
            time,
            false,                  // 是否为上边界 tick
            maxLiquidityPerTick
        );

        // 4️⃣ 更新上边界 tick 的状态
        flippedUpper = ticks.update(
            tickUpper, tick, liquidityDelta,
            _feeGrowthGlobal0X128, _feeGrowthGlobal1X128,
            secondsPerLiquidityCumulativeX128, tickCumulative,
            time,
            true,                   // 上边界 tick
            maxLiquidityPerTick
        );

        // 5️⃣ 如果 tick 状态从未初始化 -> 已初始化或反转，更新 tickBitmap
        if (flippedLower) tickBitmap.flipTick(tickLower, tickSpacing);
        if (flippedUpper) tickBitmap.flipTick(tickUpper, tickSpacing);
    }

    // 6️⃣ 计算区间 [tickLower, tickUpper] 内手续费累积
    (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) =
        ticks.getFeeGrowthInside(tickLower, tickUpper, tick, _feeGrowthGlobal0X128, _feeGrowthGlobal1X128);

    // 7️⃣ 更新 position 的流动性和手续费状态
    position.update(liquidityDelta, feeGrowthInside0X128, feeGrowthInside1X128);

    // 8️⃣ 清理 tick 数据（当流动性减少到 0 且 tick 被翻转时）
    if (liquidityDelta < 0) {
        if (flippedLower) ticks.clear(tickLower);
        if (flippedUpper) ticks.clear(tickUpper);
    }
}
```

这里的 `observeSingle` 如何获取累计值，见 [Oracle 章节](./UniswapV3-Oracle.md)。

#### 3.根据当前价格与 tick 区间的关系分三种情况计算 token 数量

这段逻辑就是上面分析的那三种数学模型，根据当前价格所处位置，在给定头寸流动性 $L$、tick 区间 $[tickLower, tickUpper]$ 时计算所需的 token 数量，`SqrtPriceMath`是 uniswapV3 底层的数学库，我们具体来看`SqrtPriceMath.getAmount0Delta`,`SqrtPriceMath.getAmount1Delta`的源码。

```solidity
    function getAmount0Delta(
        uint160 sqrtRatioAX96,
        uint160 sqrtRatioBX96,
        int128 liquidity
    ) internal pure returns (int256 amount0) {
        return
            liquidity < 0
                ? -getAmount0Delta(sqrtRatioAX96, sqrtRatioBX96, uint128(-liquidity), false).toInt256()
                : getAmount0Delta(sqrtRatioAX96, sqrtRatioBX96, uint128(liquidity), true).toInt256();
    }

    function getAmount1Delta(
        uint160 sqrtRatioAX96,
        uint160 sqrtRatioBX96,
        int128 liquidity
    ) internal pure returns (int256 amount1) {
        return
            liquidity < 0
                ? -getAmount1Delta(sqrtRatioAX96, sqrtRatioBX96, uint128(-liquidity), false).toInt256()
                : getAmount1Delta(sqrtRatioAX96, sqrtRatioBX96, uint128(liquidity), true).toInt256();
    }
```

这两个函数分别处理两种 token 在添加和移除流动性时的数量。mint 使用正的 liquidity，应付数量向上取整；burn 使用负的 liquidity，应返还数量的绝对值向下取整，再带上负号返回。下面我们看看它的下层函数是具体怎么实现的。

- 计算 $\Delta x$

$$
\Delta x = L(\frac{1}{\sqrt{P_{lower}}} - \frac{1}{\sqrt{P_{upper}}})
$$

当市场价格位于区间内部时，计算 $\Delta x$ 的下端点取当前价格 $P$。先将通用公式展开：

$$
\Delta x = \frac{L \cdot (\sqrt{P_{upper}} - \sqrt{P_{lower}})}{\sqrt{P_{lower}} \cdot \sqrt{P_{upper}}}
$$

下面的函数就是对这个公式的实现：

```solidity
function getAmount0Delta(
        uint160 sqrtRatioAX96,
        uint160 sqrtRatioBX96,
        uint128 liquidity,
        bool roundUp
    ) internal pure returns (uint256 amount0) {
        if (sqrtRatioAX96 > sqrtRatioBX96) (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);

        uint256 numerator1 = uint256(liquidity) << FixedPoint96.RESOLUTION;
        uint256 numerator2 = sqrtRatioBX96 - sqrtRatioAX96;

        require(sqrtRatioAX96 > 0);

        return
            roundUp
                ? UnsafeMath.divRoundingUp(
                    FullMath.mulDivRoundingUp(numerator1, numerator2, sqrtRatioBX96),
                    sqrtRatioAX96
                )
                : FullMath.mulDiv(numerator1, numerator2, sqrtRatioBX96) / sqrtRatioAX96;
    }
```

- 计算 $\Delta y$

$$
\Delta y = L(\sqrt{P_{upper}} - \sqrt{P_{lower}})
$$

当市场价格位于区间内部时，计算 $\Delta y$ 的上端点取当前价格 $P$，下面的函数就是对这个公式的实现：

```solidity
    function getAmount1Delta(
        uint160 sqrtRatioAX96,
        uint160 sqrtRatioBX96,
        uint128 liquidity,
        bool roundUp
    ) internal pure returns (uint256 amount1) {
        if (sqrtRatioAX96 > sqrtRatioBX96) (sqrtRatioAX96, sqrtRatioBX96) = (sqrtRatioBX96, sqrtRatioAX96);

        return
            roundUp
                ? FullMath.mulDivRoundingUp(liquidity, sqrtRatioBX96 - sqrtRatioAX96, FixedPoint96.Q96)
                : FullMath.mulDiv(liquidity, sqrtRatioBX96 - sqrtRatioAX96, FixedPoint96.Q96);
    }
```

> 💡 **总结：** `_modifyPosition`主要干了三件事：更新上下界 tick 的状态，新建一个 LP 头寸或更新 LP 头寸 position 的状态，根据当前价格所处位置计算出 LP 添加或移除流动性所需的 token 数量。学习完这个函数，我们就可以学习添加移除流动性的入口`mint`,`burn`函数了。

### mint

<img src="images/uniswapV3-07.png" alt="mint源码">

参数:

recipient：新增 Pool 头寸的 owner 地址。

tickLower：价格区间下界。

tickUpper：价格区间上界。

amount：要添加的流动性大小（目标 L）。

data：回调参数（由调用者定义，通常给 UniswapV3MintCallback 回调时用）。

return

amount0：实际需要存入的 token0 数量。

amount1：实际需要存入的 token1 数量。

`_modifyPosition` 返回本次需要支付的 token 数量，Pool 通过 `IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(amount0, amount1, data)` 回调调用者合约，由它安排付款，最后校验资金是否到账。调用者、付款方和 recipient 不必是同一地址。

使用 NPM 时，Pool 回调 NPM，由它验证池子来源并按 payer 安排付款；Pool 头寸归 NPM，NFT 则归 NPM.mint 指定的 recipient。

### burn

<img src="images/uniswapV3-08.png" alt="burn源码">

参数:

tickLower：价格区间下界。

tickUpper：价格区间上界。

amount：要移除的流动性大小（目标 L）

return

amount0：LP 应得的 token0 数量。

amount1：LP 应得的 token1 数量。

`_modifyPosition` 返回了 LP 应得的 token 数量，但是返还的 token 不会立即转给 LP，而是记在`position.tokensOwed0` / `position.tokensOwed1` 里,LP 需要自己调用`collect`函数才能真正把 token 提出来。这样做的好处是：可以批次转账 token，节省 gas，同时避免一些重入攻击。

### collect

这个函数的作用是，给 LP 调用，可以把之前通过`burn`或手续费累计到的 token，从池子合约中真正转账到用户指定的地址。

Pool.collect 只领取已经记入 tokensOwed 的数量，不主动结算新增手续费。头寸 liquidity 大于 0 时，可以先调用 `burn(tickLower, tickUpper, 0)` 刷新；[NPM.collect](./UniswapV3-NonfungiblePositionManager.md#collect) 已封装这一步。

```solidity
function collect(
    address recipient,
    int24 tickLower,
    int24 tickUpper,
    uint128 amount0Requested,
    uint128 amount1Requested
) external override lock returns (uint128 amount0, uint128 amount1) {
    // 注意这里用的是 msg.sender，所以只有 position 的 owner 才能来提取
    Position.Info storage position = positions.get(msg.sender, tickLower, tickUpper);

    // 计算实际能取出的数量：取 min(用户请求数量, position 中记录的欠款数量)
    amount0 = amount0Requested > position.tokensOwed0 ? position.tokensOwed0 : amount0Requested;
    amount1 = amount1Requested > position.tokensOwed1 ? position.tokensOwed1 : amount1Requested;

    // 如果有 token0 欠款，就减少账面 owed 并转账
    if (amount0 > 0) {
        position.tokensOwed0 -= amount0;
        TransferHelper.safeTransfer(token0, recipient, amount0);
    }

    // 如果有 token1 欠款，就减少账面 owed 并转账
    if (amount1 > 0) {
        position.tokensOwed1 -= amount1;
        TransferHelper.safeTransfer(token1, recipient, amount1);
    }
    emit Collect(msg.sender, recipient, tickLower, tickUpper, amount0, amount1);
}

```

下面是我总结的整个添加移除流动性流程图，可以串起来，再找对应的细节回顾。

<img src="images/uniswapV3-09.png" alt="流程图">

图中的 `uniswapV3MintCallback` 表示对外部调用者合约的回调；使用 NPM 时，该回调实际在 NPM 上执行。
