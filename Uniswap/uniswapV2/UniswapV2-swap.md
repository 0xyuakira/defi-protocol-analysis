# Uniswap V2 Swap

## 价格曲线

<img src="images/Uniswap01.png" alt="uniswap价格曲线" width="50%" height="50%">

_上图是忽略手续费时的储备曲线$y = k/x$，图中交易前后的点表示储备状态，沿曲线交易时 k 保持不变。曲线斜率的绝对值$y/x$对应无费边际价格，有限交易量的平均成交价格需要根据$x \cdot y = k$计算。_

### 数学计算

假设某个交易对有$x$数量的 token A 和$y$数量的 token B，有人想用$\Delta x$数量的 token A 兑换 token B，他应该得到的 token B 的数量为$\Delta y$，先忽略手续费和整数舍入，计算过程如下：

$$
(x + \Delta x)(y - \Delta y) = x y
$$

求解$\Delta y$：

$$
\Delta y = y - \frac{xy}{x + \Delta x} = \frac{y\Delta x}{x+ \Delta x}
$$

在这个公式基础上，uniswap 会对每次 swap 收取千分之 3 的手续费，这个手续费只针对用户存入的资产，所以实际到手的 token B 的数量$\Delta y$为：

$$
\Delta y = \frac{y(\Delta x \cdot 99.7\%)}{x+ (\Delta x \cdot 99.7\%)} = \frac{997y\Delta x}{1000x + 997\Delta x}
$$

由于这部分手续费留在池子中，实际储备乘积 k 会随着交易增长，为 LP 带来手续费收益；开启协议费时，其中一部分归协议，具体见 [mintFee 章节](UniswapV2-mintFee.md)。

### Price Impact

从上面含手续费的公式，我们可以得到平均成交价格，也就是每个 token A 能换到的 token B 数量（忽略整数舍入）：

$$
\frac{\Delta y}{\Delta x} = \frac{997y\Delta x}{(1000x + 997\Delta x) \cdot \Delta x} = \frac{997y}{1000x + 997\Delta x}
$$

可以看到

$$
\frac{997y}{1000x + 997\Delta x} < \frac{y}{x}
$$

在手续费率固定的情况下，交易量$\Delta x$越大，平均成交价格越低，即每个 token A 能换到的 token B 越少，这种由交易量引起的价格变化就是 price impact。表现就是“越买越贵，越卖越便宜”。

## 代码解析

<img src="images/Uniswap02.jpg" alt="uniswap源码" width="50%" height="50%">

### 1. swap 或闪电贷

```solidity
if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens
if (data.length > 0) IUniswapV2Callee(to).uniswapV2Call(msg.sender, amount0Out, amount1Out, data);
```

- `optimistically transfer tokens`,也就是池子在假设交易会成功的前提下，先进行转账，在后续再验证交易条件。
- 可以转出一种代币，也可以同时转出两种代币（双边 swap）
- 也可以作为闪电贷使用。当`data`非空时，`to`合约必须实现`uniswapV2Call`回调函数。用户可以在回调中自定义操作，并在回调返回前向 Pair 支付足够的 token，否则后续校验会 revert，整个交易回退。
- 回调合约应验证`msg.sender`是可信 Factory 创建的对应 Pair，并按业务要求限制`sender`（调用`swap`的地址），避免被伪造回调或未授权调用。

只借出并归还同一种代币时，归还量至少为$\lceil 1000 \cdot \text{借出量} / 997 \rceil$，不计整数舍入时，相对借出量的费用约为 0.3009027%。如果用另一种代币支付，则按普通 swap 的含费报价计算所需输入量。

Pair 的`lock`阻止同一个 Pair 的`mint`、`burn`、`swap`、`skim`、`sync`在调用完成前被重入，但不阻止回调操作其他 Pair。

### 2. 计算转入 token 的数量

```solidity
balance0 = IERC20(_token0).balanceOf(address(this));
balance1 = IERC20(_token1).balanceOf(address(this));
uint amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
uint amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
require(amount0In > 0 || amount1In > 0, 'UniswapV2: INSUFFICIENT_INPUT_AMOUNT');
```

这段代码计算转入 token 的数量。`_reserve0`和`_reserve1`是池子合约中记录的旧储备，`balance0`和`balance1`是转出及回调完成后的实际余额。`_reserve - amountOut`即原储备减去池子转出的数量。如果实际余额大于这个数量，差额就计为输入，否则 amountIn 等于 0。最后校验至少有一种 token 输入。Pair 不记录输入资产的专属所有者，因此普通 swap 的转账与`swap`调用需要在同一笔交易中完成，并让失败整体回退。

### 3. 校验 K 值

```solidity
uint balance0Adjusted = balance0.mul(1000).sub(amount0In.mul(3));
uint balance1Adjusted = balance1.mul(1000).sub(amount1In.mul(3));
require(balance0Adjusted.mul(balance1Adjusted) >= uint(_reserve0).mul(_reserve1).mul(1000**2), 'UniswapV2: K');
```

这段代码从实际余额中扣除输入量的 0.3% 后，检查两侧调整后余额的乘积不小于原储备乘积。使用`>=`也允许转入量超过最低所需；Pair 只校验这个下界，不会自动退还多转入的资产。Router 根据报价组织输入，并检查用户指定的最小输出或最大输入。

这里将两侧同乘$1000^2$，用整数乘法完成校验，避免除法截断影响比较结果。

### 4. 更新池子状态

<img src="images/Uniswap03.jpg" alt="uniswap源码" width="50%" height="50%">

首先检查最新的余额有没有超过池子可以记录的最大值`uint112(-1)`。然后使用旧储备更新价格累积器（见 [TWAP 章节](UniswapV2-TWAP.md)），最后将实际余额写入池子的`reserve`。
