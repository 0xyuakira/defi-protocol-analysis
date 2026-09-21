# Uniswap V2 mintFee

> 在之前的章节，我们提到过在用户发起一笔 swap 交易的时候，会从 tokenIn 中扣除千三的手续费留存在池子中。而实际上 uniswapV2 还有一个协议费机制(mintFee)，开启后，这部分手续费的六分之一，也就是交易额的 0.05%会归入协议，其余归 LP。是否开启由 Factory 的`feeTo`控制：非零地址表示开启，零地址表示关闭，具体以对应部署的链上状态为准。

### 如何收取 mintFee？

在之前的章节，我们知道 LP 每次添加和退出流动性的时候，会增发或销毁 share 代币，从而增加或减少他们对池子流动性的份额。而收取 mintFee 的时机就在这里，调用`mint`或`burn`时会先通过`_mintFee`尝试结算协议费。只有协议费已开启、`kLast != 0`、$\sqrt{k}$相对基线有所增长，且计算出的整数增发量大于 0 时，才会向`feeTo`增发对应协议份额的 share 代币。

### 数学计算

假设：

$t_1$时刻池中的流动性为$\sqrt{k_1}$

$t_2$时刻池中的流动性为$\sqrt{k_2}$

如果$t_1$到$t_2$期间没有 LP 添加或退出流动性，也没有捐赠、rebase 等非交易资金变化，忽略整数舍入，则这期间池子中累积的手续费所创造的流动性为$\sqrt{k_2}-\sqrt{k_1}$。合约本身不区分增长来源，例如捐赠经`sync`计入储备后，也可能被纳入计费。

其流动性占池子总流动性的份额为 $f = \frac{\sqrt{k_2} - \sqrt{k_1}}{\sqrt{k_2}} = 1 - \frac{\sqrt{k_1}}{\sqrt{k_2}}$

归入协议的 mintFee 占这部分流动性的六分之一，那么 mintFee 占池子总流动性的份额$f_m = \frac{1}{6} - \frac{\sqrt{k_1}}{6 \cdot \sqrt{k_2}}$

由于 LP 代币和池子中的流动性成正比，设增发前 LP 代币的总供应量为$S_t$,增发给协议的 LP 代币数量为$S_m$,则有：

$\frac{S_m}{S_t + S_m} = \frac{1}{6} - \frac{\sqrt{k_1}}{6 \cdot \sqrt{k_2}}$

求解，可得 $S_m = \frac{\sqrt{k_2} - \sqrt{k_1}}{5 \cdot \sqrt{k_2} + \sqrt{k_1}} \cdot S_t$

## mintFee 代码解析

<img src="images/Uniswap06.jpg" alt="uniswapV2 mintFee源码" width="50%" height="50%">

`_mintFee`是一个私有函数，让我们看看在哪里会调用到它

<img src="images/Uniswap07.jpg" alt="uniswap mintFee源码" width="50%" height="50%">

结合上面的代码，可以发现

- `_mintFee`只会在添加或移除流动性时被调用
- 只有`feeTo`为非零地址时才可能增发协议费份额
- 该计算实现了上面推导出的份额增发公式

`feeOn`由`feeTo != address(0)`决定，`kLast`是用于计费的储备乘积基线，清零不影响池中的实际资产。分支逻辑如下：

- **未开启协议费（`feeOn == false`）**：如果`kLast != 0`，将其清零；否则不做处理。
- **已开启协议费（`feeOn == true`）**：
  - `kLast == 0`：尚无计费基线，本次不增发协议 LP。
  - `kLast != 0`：只有`rootK > rootKLast`时才按公式计算增发量；结果大于 0，则给`feeTo`增发对应数量的 LP 代币。

`mint`和`burn`先用旧储备调用`_mintFee`结算协议份额，再读取更新后的`totalSupply`，计算并处理用户本次的增发或销毁。最后通过`_update`更新储备；若协议费开启，则将新的储备乘积记入`kLast`，作为下次计费的基线。这样本次正常添加或撤出的资产不会被当作手续费增长。`swap`会更新储备，但不更新`kLast`。
