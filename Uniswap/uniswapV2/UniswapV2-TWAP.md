# Uniswap V2 Oracle

> 价格预言机是价格的来源，uniswap 池子中两种 token 的储备量之比给出不含手续费的边际价格，其他智能合约可以通过和 uniswap 池子合约交互，从链上获取价格。

## TWAP

如果使用当前余额比例来确定价格会有什么问题？如果有人使用闪电贷进行巨额交易，从而大幅改变池子资产比例，导致价格暂时大幅度波动，然后利用另外一个使用该价格进行决策的智能合约获利。所以 uniswap 提供了一种 TWAP（Time-Weighted Average Price）的机制，价格消费者可以取某一个时间段内的价格的平均值来确定价格。攻击者若想影响平均价格，需要让异常的池内价格跨区块持续一段时间，并承担手续费成本和被套利的风险。

### 数学计算

TWAP，时间加权平均价格，是通过“时间”这个维度，将每个价格按其持续时间加权平均。

举例说明：

- 在过去一天，价格在前 12 个小时内为 10 美元，在后 12 个小时内为 11 美元，其时间加权平均价格为 $\frac{(10 \cdot 12) + (11 \cdot 12)}{12 + 12} = 10.5$
- 在过去一天，价格在前 22 个小时内为 10 美元，在后 2 个小时内为 11 美元，其时间加权平均价格为$\frac{(10 \cdot 22) + (11 \cdot 2)}{22 + 2} \approx 10.083$

如果设价格为$P$,时间长度为$T$,则 TWAP 的公式为：

$$
\frac{P_1 T_1 + P_2 T_2 + \cdots + P_n T_n}{\sum_{i=1}^{n} T_i}
$$

## 代码解析

<img src="images/Uniswap13.jpg" alt="uniswapV2 _update代码解析" width="50%" height="50%">

**从上面的公式可以看到，我们需要在每次价格变化时，累计持续时间和价格的乘积。那么 uniswap 是怎么做的呢？**

- 在 uniswap 中，`mint`、`burn`、`swap`和`sync`都会调用`_update`。它先按旧储备的价格累计经过的时间，再将当前余额写入储备；只有时间差大于零且两侧旧储备都非零时才累计价格。
- `blockTimestampLast`记录了上一次更新储备的时间戳。计算两次变化的间隔时间：`timeElapsed = blockTimestamp - blockTimestampLast`。值得注意的是，`blockTimestamp`是区块的时间戳，也就是说，同一区块内当多次调用`_update`函数时，只有第一次才会按更新前的储备价格累计。不过，TWAP 并不能完全避免价格操纵。在低流动性、低活跃度的交易对中，攻击者可能以较低成本维持跨区块的异常价格。在采用中心化排序器的 L2 上，若攻击者能够影响交易排序，风险还可能进一步增加。实际使用时，应兼顾价格时效性，适当延长 TWAP 窗口，并结合 Chainlink 等独立价格源交叉验证，降低风险。
- 价格累计的值`price0CumulativeLast`和`price1CumulativeLast`是 uint 类型，一个 slot 是 32 个字节，也就是它们最大可以表示为
  $2^{256} - 1$。当超过这个值时，会发生溢出。Uniswap V2 使用 Solidity 0.5.16，这里的加减法不会因为溢出而 revert，而是按模$2^{256}$回绕。
  - 假设，token0 上一次累计的价格是$2^{256} - 10$，当前累计价格是$2^{256} + 10$。从数学运算上讲，它们的差值是 20。
  - 但是实际上当前累计价格因为溢出，会记录为 10。实际计算：$10 - (2^{256} - 10)$，结果为$-2^{256} + 20$。按模$2^{256}$回绕后，结果还是 20，符合数学运算的结果。
- `UQ112x112.encode(_reserve1).uqdiv(_reserve0)`使用 UQ112.112 定点数，先将`_reserve1`乘以$2^{112}$，再除以`_reserve0`并向下取整。缩放因子为$2^{112}$，分辨率为$2^{-112}$，这样可以保留更多小数精度。

**从上面代码可以看到 uniswapV2 的代码中，只是不停的记录了累计价格，那我们怎么计算我们想要的平均价格呢？**

如果我们想获取$t_{n-1}$到$t_n$的加权平均价格，假设已记录这两个时刻的累计价格$p_{n-1}$和$p_n$，则可以通过下面这个公式：

$$
\frac{p_n - p_{n-1}}{t_n - t_{n-1}}
$$

每个快照都需要同时记录累计价格和对应时刻。直接读取 Pair 的`price0CumulativeLast`或`price1CumulativeLast`时，它们对应的是`getReserves`返回的`blockTimestampLast`，不能直接配上读取时的当前时间。Pair 只保存最新值，历史观测需要由价格消费者自行保存。

链上时间差按 uint32 模$2^{32}$计算，累计价格差按 uint256 模$2^{256}$计算。实际观测间隔应满足$0 < t_n - t_{n-1} < 2^{32}$秒，且窗口内两侧储备均非零；零储备期间不累计价格，不能直接按完整窗口时长求平均。迁移至 Solidity 0.8+ 时，需要用`unchecked`保留这些加减法的模运算语义。将链上累计值代入公式后，结果仍带有$2^{112}$的缩放。

**如果最后一次快照是三小时之前的快照怎么办？**
如果过去三小时没有触发`_update`，直接读取累计值仍会得到三小时前的旧值。可以先调用`sync`将累计值推进到当前时刻，再读取累计值和对应时间戳：

```solidity
function sync() external lock {
    _update(IERC20(token0).balanceOf(address(this)), IERC20(token1).balanceOf(address(this)), reserve0, reserve1);
}
```

也可以在两侧储备非零时，使用[官方 OracleLibrary 的`currentCumulativePrices`](https://github.com/Uniswap/v2-periphery/blob/master/contracts/libraries/UniswapV2OracleLibrary.sol)，只读补算到当前时刻并返回对应时间戳，无需调用`sync`或写入 Pair。

**为什么 TWAP 分别跟踪两种 token 的累计价格？**

token0 的价格实际上是 token1 的数量和 token0 数量的比率，反之亦然。实际上两种价格就是分子和分母颠倒的数字。但是当我们累计价格时，就不能通过反转其中一个累计价格来获得另外一个累计价格了，比如：

- token0 的累计价格：$2 + 3$
- token1 反转后：$\frac{1}{2 + 3}$
- token1 实际的累计价格：$\frac{1}{2} + \frac{1}{3}$

这里的储备以 token 最小单位计量。若要得到完整 token1/token0 的价格，应将`price0`方向的定点平均值乘以$10^{decimals_0-decimals_1}/2^{112}$；换算时保留小数精度，`price1`方向相反。
