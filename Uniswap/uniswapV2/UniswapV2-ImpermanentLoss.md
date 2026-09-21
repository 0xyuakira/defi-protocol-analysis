# Uniswap V2 Impermanent Loss

### 什么是无常损失？

无常损失，是指当池中两种资产的相对价格偏离初始值时，交易和套利会改变池中的资产配比。在不计手续费的情况下，LP 的资产总价值会低于单纯持有初始资产的总价值，这部分差额就是“机会成本”上的损失。

### 举例：

以下例子和推导忽略手续费与整数舍入，假设初始投入后没有额外的流动性变动或捐赠，并且池内价格经套利与外部相对价格一致。

在一个 ETH/USDC 的交易对中，池子当时的价格为 100 USDC/ETH ，LP 添加了 1 ETH 和 100 USDC 进入流动性池

这部分流动性为： $\sqrt{1 \cdot 100} = 10$

此时 LP 拥有的价值为：200 USDC

- **ETH 增值**

  ETH 增值到 156.25 USDC/ETH

  由于 LP 的流动性不变， $\sqrt{ 0.8 * 125} = 10$

  此时 LP 拥有代币的数量为 ：$0.8 ETH + 125 USDC$

  LP 拥有的价值： $0.8 \cdot 156.25 + 125 = 250 USDC$

  假设 LP 最初没有添加流动性，而是持有这些代币，那么此时这些代币的总价值为：$1 \cdot 156.25 + 100 = 256.25 USDC$
  **添加流动性比不添加流动性少赚了 ：256.25 - 250 = 6.25 USDC**

- **ETH 贬值**

  ETH 贬值到：64 USDC/ETH

  由于 LP 流动性不变， $\sqrt{1.25 * 80} = 10$

  此时 LP 拥有的代币数量为： $1.25 ETH + 80 USDC$

  LP 拥有的价值： $1.25 \cdot 64 + 80 = 160 USDC$

  假设 LP 最初没有添加流动性，而是持有这些代币，那么此时这些代币的总价值为：$1 \cdot 64 + 100 = 164 USDC$
  **添加流动性比不添加流动性多亏了： 164 - 160 = 4 USDC**

💡 在上述假设下，相对价格无论上涨还是下跌，偏离初始价格都会产生无常损失；回到初始价格时，无常损失为零。实际交易中的手续费收益可以抵消部分甚至全部损失，但不保证足额补偿。

### 数学模型

假设现在有一个交易对，其中两种 token 的数量分别是$x$,$y$，价格为$p$，$k$的平方根为$L$：

$$
p = \frac{y}{x}
$$

$$
L = \sqrt{x \cdot y}
$$

这里的$L$是流动性的数学度量，不代表实际铸造的 LP Token 数量。

由上面两个公式很容易得出：

$$
y = L \cdot \sqrt{p}
$$

$$
x = \frac{L}{\sqrt{p}}
$$

假设在$t_0$时刻的价格为$p_0$，$t_1$时刻的价格为$p_1$，$p_0$和$p_1$的变化关系为$p_1 = p_0 \cdot d$

**如果用户在$t_0$时刻持有 token，在$t_1$时刻持有这些 token 的价值为$v_{hold}$:**

$$
\begin{aligned}
v_{hold} &= y_0 + x_0 \cdot p_1 \\
         &= L \cdot \sqrt{p_0} + \frac{L}{\sqrt{p_0}} \cdot p_0 \cdot d \\
         &= L \cdot \sqrt{p_0} + L \cdot \sqrt{p_0} \cdot d \\
         &= (1 + d) \cdot L \cdot \sqrt{p_0}
\end{aligned}
$$

**如果用户在$t_0$时刻将 token 添加流动性，在$t_1$时刻持有这些 token 的价值为$v_1$:**

$$
\begin{aligned}
v_1 &= y_1 + x_1 \cdot p_1 \\
    &= L \cdot \sqrt{p_1} + \frac{L}{\sqrt{p_1}} \cdot p_1 \\
    &= 2 \cdot L \cdot \sqrt{p_1} \\
    &= 2 \cdot L \cdot \sqrt{p_0 \cdot d}
\end{aligned}
$$

**用户的无常损失 IL 为:**

$$
\begin{aligned}
IL &= \frac{v_1 - v_{hold}}{v_{hold}} \\
   &= \frac{2 \cdot L \cdot \sqrt{p_0 \cdot d} - (1 + d) \cdot L \cdot \sqrt{p_0}}{(1 + d) \cdot L \cdot \sqrt{p_0}} \\
   &= \frac{2 \cdot \sqrt{d}}{1 + d} - 1
\end{aligned}
$$

**函数的对称性**

该函数具有倒数的对称性，也就是 $IL(d) = IL(\frac{1}{d})$，证明如下：

$$
\begin{aligned}
IL\left(\frac{1}{d}\right)
&= \frac{2 \cdot \sqrt{\frac{1}{d}}}{1 + \frac{1}{d}} - 1\\
&= \frac{2 \cdot \frac{1}{\sqrt{d}}}{\frac{d + 1}{d}} - 1\\
&= \frac{2}{\sqrt{d}} \cdot \frac{d}{d + 1} - 1\\
&= \frac{2 \sqrt{d}}{d + 1} - 1\\
&= IL(d)
\end{aligned}
$$

从上面的公式可以看出，无常损失函数具有以下特征：

- **对称性**，$IL(d) = IL(\frac{1}{d})$，价格比率互为倒数时，无常损失相同。例如价格变为原来的 10 倍，与跌到原来的十分之一，无常损失相同。
- **非正性**，对于$d > 0$，总有$IL(d) \le 0$，且只有$d = 1$时，无常损失为 0。
