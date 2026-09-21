# Uniswap V2

### Constant Product Automated Market Maker

AMM(自动化做市商)有许多不同的构建方法，Uniswap V2 使用了**恒定乘积做市商**来构建。其核心数学原理只是一个非常简单的公式：

**$x * y = k$**

- x 和 y 代表了一个交易对中两种 token 的储备量，k 是它们的乘积。当用户进行交易时，将他们打算卖出的 token 放进池子，将他们打算购买的 token 移除池子，这会改变池子中两种 token 的数量。在忽略手续费的理想报价模型中，**每次交易前后，k 保持不变**。

### AMM 的优点

- 在 AMM 中，价格发现是自动的。它由池中资产比例决定，无需像订单簿模型中等待合适的“出价”和“要价”，在储备非零且交易满足合约约束时，可以按曲线报价。实际平均成交价格还会受到交易量和手续费的影响。

- 在 AMM 中，只需要记录两个 token，并按照一定的规则转移它们，所以与需要大量簿记的订单簿相比，更加节省 gas。

### AMM 的缺点

- **价格总是变动，滑点频繁**
  在 AMM 中，价格发现是由池中资产比例决定的，所以每一笔交易都会影响价格。买入或卖出通常会遇到更多的滑点，受流动性深度，大额交易，价格波动，交易顺序等因素影响。攻击者可能会利用滑点发动三明治攻击。

- **LP 会遭受无常损失**
  当池中两种资产的相对价格偏离初始值时，LP 会面临无常损失，具体计算过程见无常损失章节。

### Uniswap V2 的架构

Uniswap 使用了`core-periphery`的设计模式，最核心的逻辑位于 core 中，可选逻辑位于 periphery。这样做的目的是让 core 包含尽可能少的代码，减少核心业务逻辑中出现错误的可能性。用户可以选择通过 periphery 中的合约与 core 中的合约进行交互，也可以自己定制逻辑通过自己的合约直接与 core 中的合约交互。

# core

- **UniswapV2Factory**
  1. 负责创建 UniswapV2Pair，通过`create2`的方式创建。并在内部保存了所有 UniswapV2Pair 的地址。
  2. 保存了 feeTo 这个可以收取协议费的地址，以及拥有设置并修改 feeTo 权限的 feeToSetter 地址。
- **UniswapV2Pair**
  1. 该合约持有两个 ERC20 代币的交易对，每个交易对都有一个 UniswapV2Pair 合约。如果所需的交易对不存在，则无需许可从 UniswapV2Factory 创建一个新的合约。
  2. 交易者可以交换，LP 可以为其提供流动性。UniswapV2Pair 合约本身也是 ERC20 代币，该代币为 LP Token，用于记录用户的流动性份额，具体见 [mint/burn 章节](UniswapV2-mintAndburn.md#追踪-lp-的流动性份额)。

阅读 Pair 代码时，需要区分以下状态：

| 状态 | 含义 |
| --- | --- |
| `reserve0`、`reserve1` | 上次`_update`写入的两种 token 储备量 |
| `token.balanceOf(pair)` | token 合约中记录的 Pair 当前实际余额，直接转账或 rebase 可能使它与储备量不同 |
| `totalSupply`、用户 LP 余额 | LP Token 总量与用户持有量；用户份额为用户 LP 余额除以`totalSupply` |
| `kLast` | 协议费使用的储备乘积基线，具体见 [mintFee 章节](UniswapV2-mintFee.md) |
| `price0CumulativeLast`、`price1CumulativeLast`、`blockTimestampLast` | 两个方向的价格累计值，以及上次更新储备的时间戳，具体见 [TWAP 章节](UniswapV2-TWAP.md) |

`sync()`将实际余额写入储备；`skim(to)`将实际余额超过储备的部分转给`to`，不更新储备。两者都可以由任何人调用，Pair 不会为直接转入的资产记录专属存款人。

# periphery

提供了 2 个 router 合约，这些合约提供一些面向用户的机制，在 uniswap 的基本逻辑上做了增强，使得用户与 uniswap 交互更加安全。Library 根据储备计算报价，Router 将转账和 Pair 调用组合在同一笔交易中，并检查用户设置的成交界限。Pair 负责校验结算后的不变量，不判断外部公平价格；这些界限是否合理，仍需调用者确定。
