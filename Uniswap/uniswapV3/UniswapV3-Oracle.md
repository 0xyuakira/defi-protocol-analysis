# Uniswap V3 Oracle

> V2 累加价格，V3 累加价格的对数索引 tick，并在 Pool 中保存历史观察值。外部合约通过两次累计值的差，就能计算一个时间窗口内的平均价格。

## Observation

每个 Pool 通过 Oracle 库维护观察值：

```solidity
struct Observation {
    uint32 blockTimestamp;
    int56 tickCumulative;
    uint160 secondsPerLiquidityCumulativeX128;
    bool initialized;
}
```

- blockTimestamp：本次观察的时间戳。
- tickCumulative：从初始化开始，按时间累计的 tick，即 $\sum tick_j \cdot \Delta t_j$；tick 为负时，累计值也会减少。
- secondsPerLiquidityCumulativeX128：累计的“时间 / 活跃流动性”，放大 $2^{128}$ 倍保存。
- initialized：该槽位是否已经写入有效观察值。

改变 tick 的 swap，以及改变活跃流动性的 mint/burn，会调用 observations.write，用变更前的 tick 和 L 累加已经经过的时间。同一时间戳只写一次。具体逻辑见 [Pool](https://github.com/Uniswap/v3-core/blob/d0831dc6b8a318df3872b6d68f6de135c9f3ec29/contracts/UniswapV3Pool.sol) 和 [Oracle.transform / write](https://github.com/Uniswap/v3-core/blob/d0831dc6b8a318df3872b6d68f6de135c9f3ec29/contracts/libraries/Oracle.sol)。

## observe 与平均价格

假设要查询最近 T 秒的平均价格，且 $T>0$，可以向 pool.observe 传入 [T,0]：

```solidity
uint32[] memory secondsAgos = new uint32[](2);
secondsAgos[0] = T; // T 秒前
secondsAgos[1] = 0; // 当前
(int56[] memory tickCumulatives, uint160[] memory secondsPerLiquidityCumulativeX128s) =
    pool.observe(secondsAgos);
```

记 tick 累计值为 $C$，当前时间为 $t$，则未取整的平均 tick 为：

$$
\bar{i}=\frac{C(t)-C(t-T)}{T}
$$

[OracleLibrary.consult](https://github.com/Uniswap/v3-periphery/blob/0682387198a24c7cd63566a2c58398533860a5d1/contracts/libraries/OracleLibrary.sol) 返回整数平均 tick，即 $\lfloor\bar{i}\rfloor$。Solidity 的有符号整数除法向零取整，因此累计差为负且除不尽时，还要减 1。例如 $-5/3$ 应得到 -2，而不是 -1。

为什么平均 tick 可以用来求平均价格？把各段 tick 对应的价格记为 $P_j=1.0001^{tick_j}$，就有：

$$
1.0001^{\bar{i}}
=1.0001^{\sum_j tick_j\Delta t_j/T}
=\prod_j P_j^{\Delta t_j/T}
$$

因此它对应 **tick 刻度价格的时间加权几何平均**。实际用 $1.0001^{\lfloor\bar{i}\rfloor}$ 报价，还会有整数 tick 和定点计算的取整误差。这里的价格方向和 decimals 换算沿用 [Tick 章节](./UniswapV3-tick.md)，不能直接当作完整 token 的报价。

## 观察容量与历史长度

slot0 中有三个相关字段：

| 字段 | 含义 |
|---|---|
| observationIndex | 最近一次写入的观察值索引 |
| observationCardinality | 当前环形数组使用的容量，扩容后其中可能还有未初始化的槽位 |
| observationCardinalityNext | 计划扩展到的容量 |

Pool 初始化时只写入一个观察值。任何人都可以调用 increaseObservationCardinalityNext 扩容，最多 65535 个槽位；扩容不会补出过去的数据，需要后续交互继续写入。

数组写满后循环覆盖旧记录，所以容量不等于固定的历史时长。可查询范围取决于最早的有效观察值；目标时间比它更早时，observe 会以 OLD 回退。

如果目标时间位于两个观察值之间，Oracle 会通过插值得到累计值；如果比最新观察值更晚，则用当前 tick 和活跃 L 外推。即使池子一段时间没有交互，observe([T,0]) 也能计算到当前时间，但前提仍是 T 秒前处于可查询范围内。这个查询只在内存中计算，不会写入新快照。见 [Oracle.observeSingle](https://github.com/Uniswap/v3-core/blob/d0831dc6b8a318df3872b6d68f6de135c9f3ec29/contracts/libraries/Oracle.sol)。

## 平均流动性

价格累计和流动性累计使用不同的口径。记 secondsPerLiquidityCumulativeX128 为 $S$，忽略定点舍入：

$$
\frac{S(t)-S(t-T)}{2^{128}}
\approx \sum_j \frac{\Delta t_j}{\max(1,L_j)}
$$

因此得到的是活跃流动性的时间加权**调和平均**：

$$
L_{\text{harmonic}}
=\frac{T}{\sum_j \Delta t_j/\max(1,L_j)}
\approx \frac{T\cdot2^{128}}{S(t)-S(t-T)}
$$

这里的 L 是池子的活跃流动性，不是全部头寸的 L 之和。L 为 0 时，源码用 1 作为分母避免除零，所以严格说累计的是 $\max(1,L)$ 的倒数，不能把它当作池子始终存在流动性的证明。

OracleLibrary.consult 使用 $T(2^{160}-1)/(\Delta S\cdot2^{32})$ 并向下取整实现这一计算，细节与上面的理想式略有差别。

## 使用时需要注意

TWAP 可以降低短时价格变化的影响，但不保证价格不可操纵，低流动性池尤其需要谨慎。拉长窗口通常会增加操纵成本，也会让报价更滞后；实际使用还应检查历史覆盖和流动性，并结合其他价格源。可以继续参考 [官方 Oracle 文档](https://developers.uniswap.org/docs/protocols/v3/concepts/price-oracles)。
