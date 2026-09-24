# Uniswap V3 Swap

在 V2 中，swap 逻辑非常简洁：池子合约维护了两种资产储备数量，利用恒定乘积公式 $x \cdot y = k$ ，就可以一步推导出任意一次兑换的结果。但是在 V3 中，流动性不是全局恒定的，而是随着价格区间的不同分段变化的。Pool 不再像 V2 一样维护`reserve0/reserve1`，区间报价主要使用的状态变量是：

- liquidity ：当前价格所在的 tick 区间的流动性
- sqrtPriceX96 ：当前价格 $\sqrt{P}$的定点数
- liquidityNet : 每个 tick 上的流动性变化量

因此，V3 的 swap 不再像 V2 那样一步套公式，而是一个逐区间推进的过程：

- 在一个区间内，使用当前 $\sqrt{P}$ 和当前价格所在 tick 区间的流动性 $L$ 之间的公式计算兑换
- 当价格将要跨越 tick 时，再根据 liquidityNet 计算下一个区间的流动性 $L$ ,继续下一段的计算

这样设计的好处也是显而易见的：

- 区间内的价格推进只需更新 $\sqrt{P}$，不需要维护 V2 那样的储备变量
- 跨区间的时候，只需要计算下一个区间的流动性，而不是调整每个区间的 token 数量
- 上面两点不仅仅减少了合约复杂度，同时也节省了大量 gas，而区间内的兑换数量可以由价格变化和 $L$ 推导；当前价格和活跃 $L$ 不能还原全池真实余额

## 数学模型

<img src="images/sadkmska.png" alt="uniswapV3 swap模型" width="50%" height="50%">

因为 V3 的 swap，是一个逐区间推进的过程，每一段 tick 区间内的数学模型都是一样的，所以我们分析 swap 的数学模型的前提是在同一个区间内 swap。假设存在交易对 (x,y)，价格取 $P=y/x$，价格区间是 $[P_{lower},P_{upper}]$，此区间的活跃流动性 $L>0$ 且保持不变。以下忽略整数舍入，输入数量均指扣除手续费后的净输入。

1. 卖出 y，得到 x

当前市场价格为 $P_b$，卖出的 y token 扣除手续费后净输入为 $\Delta y$，计算我们该得到多少数量的 x token？

假设这笔净输入使市场价格上涨至 $P_a$，则有：

$$
\Delta y = y_a - y_b = L(\sqrt{P_a} - \sqrt{P_b})
$$

已知： $\Delta y$ ， $P_b$ ， $L$ ，求 $\sqrt{P_a}$ ：

$$
\sqrt{P_a} = \frac{\Delta y}{L} + \sqrt{P_b}
$$

得到 $\sqrt{P_a}$ ，则可以计算我们得到的 x token 的数量 $\Delta x$ ：

$$
\Delta x = x_b - x_a = L(\frac{1}{\sqrt{P_b}} - \frac{1}{\sqrt{P_a}})
$$

2. 卖出 x，得到 y

当前市场价格为 $P_a$，卖出的 x token 扣除手续费后净输入为 $\Delta x$，计算我们该得到多少数量的 y token？

假设这笔净输入使市场价格下降至 $P_b$，则有：

$$
\Delta x = x_b - x_a = L(\frac{1}{\sqrt{P_b}} - \frac{1}{\sqrt{P_a}})
$$

已知： $\Delta x$ ， $P_a$ ， $L$ ，求 $\sqrt{P_b}$ :

$$
\sqrt{P_b} = \frac{L\sqrt{P_a}}{\sqrt{P_a}\Delta x + L}
$$

得到 $\sqrt{P_b}$ , 则可以计算我们得到的 y token 的数量 $\Delta y$ ：

$$
\Delta y = y_a - y_b = L(\sqrt{P_a} - \sqrt{P_b})
$$

上面的推导针对指定输入：先用费后净输入推算新价格，再计算应该支付给交易者多少 token。指定输出则从输出数量反推新价格和所需净输入，再计入手续费。如果剩余数量需要跨越当前区间，就先结算到边界，再在下一 tick 区间重复同样的步骤，直到数量处理完或到达价格限制。

## 源码实现

### 1. 函数入参

```solidity
function swap(
  address recipient,
  bool zeroForOne,
  int256 amountSpecified,
  uint160 sqrtPriceLimitX96,
  bytes calldata data
)
```

- recipient：交易接收地址
- zeroForOne：交易方向，`true`表示 token0 -> token1,`false`表示 token1 -> token0
- amountSpecified：指定交易数量， $>0$ 表示指定输入， $<0$ 表示指定输出
- sqrtPriceLimitX96：交易允许到达的平方根价格边界，合约不会把价格推进超过这个值
- data：传递给回调`uniswapV3SwapCallback`的字节

### 2. 前置检查

```solidity
        require(amountSpecified != 0, 'AS');

        Slot0 memory slot0Start = slot0;

        require(slot0Start.unlocked, 'LOK');

        require(
            zeroForOne
                ? sqrtPriceLimitX96 < slot0Start.sqrtPriceX96 && sqrtPriceLimitX96 > TickMath.MIN_SQRT_RATIO
                : sqrtPriceLimitX96 > slot0Start.sqrtPriceX96 && sqrtPriceLimitX96 < TickMath.MAX_SQRT_RATIO,
            'SPL'
        );
        slot0.unlocked = false;
```

- 根据交易方向，校验 sqrtPriceLimitX96 参数
- `slot0.unlocked`是一个全局状态变量，用来防止回调时被重入攻击

### 3. 创建缓存

```solidity
SwapCache memory cache = SwapCache({
    liquidityStart: liquidity,
    blockTimestamp: _blockTimestamp(),
    feeProtocol: zeroForOne ? (slot0Start.feeProtocol % 16) : (slot0Start.feeProtocol >> 4),
    secondsPerLiquidityCumulativeX128: 0,
    tickCumulative: 0,
    computedLatestObservation: false
});
```

`SwapCache` 是 swap 开始时的一些环境快照，用来避免在循环里重复读取链上存储。

- liquidityStart：swap 开始时的活跃流动性
- blockTimestamp：当前区块时间戳
- feeProtocol：协议费的抽成分母，0 表示关闭，n 表示从手续费中抽取 1/n。输入为 token0 时取低四位，输入为 token1 时取高四位
- secondsPerLiquidityCumulativeX128：初始化为 0，首次跨越已初始化 tick 时计算并缓存截至当前时间的单位流动性秒数累计值
- tickCumulative：初始化为 0，与上项一起计算并缓存 tick 的时间累计值
- computedLatestObservation：本次 swap 是否已缓存上述累计值，首次计算后设为 true，后续跨界复用；这里还没有写入 observation

### 4. 创建状态机

```solidity
bool exactInput = amountSpecified > 0;

SwapState memory state =
    SwapState({
        amountSpecifiedRemaining: amountSpecified,
        amountCalculated: 0,
        sqrtPriceX96: slot0Start.sqrtPriceX96,
        tick: slot0Start.tick,
        feeGrowthGlobalX128: zeroForOne ? feeGrowthGlobal0X128 : feeGrowthGlobal1X128,
        protocolFee: 0,
        liquidity: cache.liquidityStart
    });
```

`SwapState` 是在 while 循环中不断更新的状态机。

- amountSpecifiedRemaining：剩余待处理的交易数量，指定输入时为正，指定输出时为负；成交时绝对值向 0 收敛
- amountCalculated：已经计算出的对手 token 的累计数量。最终会作为 swap 的结果返回。
- sqrtPriceX96：当前市场 $\sqrt{P}$ 的定点数
- tick：当前 tick，随着跨越 tick 更新
- feeGrowthGlobalX128：当前交易方向对应的全局每单位流动性产生的累计手续费，如果是 token0 -> token1，取`feeGrowthGlobal0X128`，如果是 token1 -> token0，取`feeGrowthGlobal1X128`
- protocolFee：本次 swap 累计产生的协议费用
- liquidity：当前市场价格所在 tick 区间的流动性，会在跨越 tick 时更新

### 5. 逐 tick 推进 swap

```solidity
    while (state.amountSpecifiedRemaining != 0 && state.sqrtPriceX96 != sqrtPriceLimitX96) {}
```

这段 while 循环代码正是 swap 的核心逻辑，直到`amountSpecifiedRemaining`被消耗完，或者价格到达`sqrtPriceLimitX96`限制。到达价格限制时，Pool 可能只完成部分兑换，实际输入和输出以返回值为准。

```solidity
            StepComputations memory step;

            step.sqrtPriceStartX96 = state.sqrtPriceX96;

            (step.tickNext, step.initialized) = tickBitmap.nextInitializedTickWithinOneWord(
                state.tick,
                tickSpacing,
                zeroForOne
            );

            if (step.tickNext < TickMath.MIN_TICK) {
                step.tickNext = TickMath.MIN_TICK;
            } else if (step.tickNext > TickMath.MAX_TICK) {
                step.tickNext = TickMath.MAX_TICK;
            }

            step.sqrtPriceNextX96 = TickMath.getSqrtRatioAtTick(step.tickNext);

```

- 初始化`step`，存放这一轮 swap 的中间状态
- `nextInitializedTickWithinOneWord`按照当前交易方向在一个 word 内查找已初始化 tick；未找到时返回该次查询的 word 边界和`initialized=false`，再将结果裁剪到`[MIN_TICK, MAX_TICK]`范围内
- 计算本次返回的 tick 的 sqrtPriceX96

```solidity
(state.sqrtPriceX96, step.amountIn, step.amountOut, step.feeAmount) = SwapMath.computeSwapStep(
                state.sqrtPriceX96,
                (zeroForOne ? step.sqrtPriceNextX96 < sqrtPriceLimitX96 : step.sqrtPriceNextX96 > sqrtPriceLimitX96)
                    ? sqrtPriceLimitX96
                    : step.sqrtPriceNextX96,
                state.liquidity,
                state.amountSpecifiedRemaining,
                fee
            );
```

`SwapMath.computeSwapStep`实现单步计算，目标价格取沿交易方向更近的 tick 边界或`sqrtPriceLimitX96`。指定输入时，先扣除手续费，再判断净输入能否到达目标；指定输出时，先比较剩余输出需求与到达目标可支付的数量，再反推价格和所需输入。数量不足以到达目标时，在区间内结束本步；到达 tick 边界且仍有剩余数量时继续推进；到达价格限制时结束 swap。

数量计算中，可用净输入向下取整，应收的`amountIn`向上取整，可付的`amountOut`向下取整，指定输出还会限制`amountOut`不超过剩余需求。指定输入且未到目标价格时，剩余输入与实际`amountIn`的差额作为本步手续费；其余分支按`amountIn * feePips / (1e6 - feePips)`向上取整计算手续费。

遇到 $L=0$ 的空区间时，不能直接套用上面除以 $L$ 的公式。此时本步输入、输出和手续费均为 0，价格推进到目标，继续查找后续流动性，或在到达价格限制时结束。

```solidity
            if (exactInput) {
                state.amountSpecifiedRemaining -= (step.amountIn + step.feeAmount).toInt256();
                state.amountCalculated = state.amountCalculated.sub(step.amountOut.toInt256());
            } else {
                state.amountSpecifiedRemaining += step.amountOut.toInt256();
                state.amountCalculated = state.amountCalculated.add((step.amountIn + step.feeAmount).toInt256());
            }
```

更新状态机中的剩余待处理交易数量和已经计算出的对手 token 的数量。

```solidity
            if (cache.feeProtocol > 0) {
                uint256 delta = step.feeAmount / cache.feeProtocol;
                step.feeAmount -= delta;
                state.protocolFee += uint128(delta);
            }

            // update global fee tracker
            if (state.liquidity > 0)
                state.feeGrowthGlobalX128 += FullMath.mulDiv(step.feeAmount, FixedPoint128.Q128, state.liquidity);
```

- 从手续费中计算出协议费的抽成
- 更新全局手续费累计

```solidity
if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
                // if the tick is initialized, run the tick transition
                if (step.initialized) {
                    // check for the placeholder value, which we replace with the actual value the first time the swap
                    // crosses an initialized tick
                    if (!cache.computedLatestObservation) {
                        (cache.tickCumulative, cache.secondsPerLiquidityCumulativeX128) = observations.observeSingle(
                            cache.blockTimestamp,
                            0,
                            slot0Start.tick,
                            slot0Start.observationIndex,
                            cache.liquidityStart,
                            slot0Start.observationCardinality
                        );
                        cache.computedLatestObservation = true;
                    }
                    int128 liquidityNet =
                        ticks.cross(
                            step.tickNext,
                            (zeroForOne ? state.feeGrowthGlobalX128 : feeGrowthGlobal0X128),
                            (zeroForOne ? feeGrowthGlobal1X128 : state.feeGrowthGlobalX128),
                            cache.secondsPerLiquidityCumulativeX128,
                            cache.tickCumulative,
                            cache.blockTimestamp
                        );
                    // if we're moving leftward, we interpret liquidityNet as the opposite sign
                    // safe because liquidityNet cannot be type(int128).min
                    if (zeroForOne) liquidityNet = -liquidityNet;

                    state.liquidity = LiquidityMath.addDelta(state.liquidity, liquidityNet);
                }

                state.tick = zeroForOne ? step.tickNext - 1 : step.tickNext;
            } else if (state.sqrtPriceX96 != step.sqrtPriceStartX96) {
                // recompute unless we're on a lower tick boundary (i.e. already transitioned ticks), and haven't moved
                state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
            }
```

- 如果价格到达 tick 边界：

  只有边界已初始化时，才调用`ticks.cross`翻转 outside 累计值并取回`liquidityNet`，再由调用方按穿越方向调整符号、更新活跃流动性。

  向右到达边界 i 时记录 tick=i，向左时记录 tick=i−1，表示跨界后所在的一侧。因此，最终记录的`slot0.tick`不一定等于`getTickAtSqrtRatio(sqrtPriceX96)`，不能直接用后者替换。

- 如果价格没到边界且发生了变化：

  根据新价格反推当前 tick

#### 更新 slot0 状态

```solidity
 if (state.tick != slot0Start.tick) {
            (uint16 observationIndex, uint16 observationCardinality) =
                observations.write(
                    slot0Start.observationIndex,
                    cache.blockTimestamp,
                    slot0Start.tick,
                    cache.liquidityStart,
                    slot0Start.observationCardinality,
                    slot0Start.observationCardinalityNext
                );
            (slot0.sqrtPriceX96, slot0.tick, slot0.observationIndex, slot0.observationCardinality) = (
                state.sqrtPriceX96,
                state.tick,
                observationIndex,
                observationCardinality
            );
        } else {
            // otherwise just update the price
            slot0.sqrtPriceX96 = state.sqrtPriceX96;
        }

        // update liquidity if it changed
        if (cache.liquidityStart != state.liquidity) liquidity = state.liquidity;

        // update fee growth global and, if necessary, protocol fees
        // overflow is acceptable, protocol has to withdraw before it hits type(uint128).max fees
        if (zeroForOne) {
            feeGrowthGlobal0X128 = state.feeGrowthGlobalX128;
            if (state.protocolFee > 0) protocolFees.token0 += state.protocolFee;
        } else {
            feeGrowthGlobal1X128 = state.feeGrowthGlobalX128;
            if (state.protocolFee > 0) protocolFees.token1 += state.protocolFee;
        }
```

循环结束后，将活跃 liquidity、当前价格和 tick 写入全局状态，并更新手续费累计和协议费用。如果 tick 发生变化，还会调用`observations.write`，用交易前的 tick 和 liquidity 累计到当前时间；同一时间戳不重复写入。预言机的完整说明见 [Oracle](./UniswapV3-Oracle.md)。

#### 根据交易方向返回 amount0，amount1，转账或回调支付

```solidity
        (amount0, amount1) = zeroForOne == exactInput
            ? (amountSpecified - state.amountSpecifiedRemaining, state.amountCalculated)
            : (state.amountCalculated, amountSpecified - state.amountSpecifiedRemaining);

        // do the transfers and collect payment
        if (zeroForOne) {
            if (amount1 < 0) TransferHelper.safeTransfer(token1, recipient, uint256(-amount1));

            uint256 balance0Before = balance0();
            IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(amount0, amount1, data);
            require(balance0Before.add(uint256(amount0)) <= balance0(), 'IIA');
        } else {
            if (amount0 < 0) TransferHelper.safeTransfer(token0, recipient, uint256(-amount0));

            uint256 balance1Before = balance1();
            IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(amount0, amount1, data);
            require(balance1Before.add(uint256(amount1)) <= balance1(), 'IIA');
        }

        emit Swap(msg.sender, recipient, amount0, amount1, state.sqrtPriceX96, state.liquidity, state.tick);
        slot0.unlocked = true;
```
