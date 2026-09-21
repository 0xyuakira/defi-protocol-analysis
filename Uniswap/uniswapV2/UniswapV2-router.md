# UniswapV2 router

> `periphery`中提供了两个重要的合约`UniswapV2Router01`和`UniswapV2Router02`。其中后者延续了前者的接口和功能，并增加了对 fee on transfer token 的支持。

## router 合约是面向用户的合约，用于

- 安全的添加和移除流动性
- 安全的 swap
- 添加了`core`合约中省略的与滑点相关的安全检查
- 通过与`WETH`合约集成，增加交换以太币的能力
- 增加了对`Fee on Transfer Tokens`的支持

## swapExactTokensForTokens 和 swapTokensForExactTokens

<img src="images/Uniswap09.jpg" alt="uniswapV2 router源码" width="50%" height="50%">

### 参数

- `swapExactTokensForTokens`：意味着正在交换的输入 token 的数量是固定的。用户需要准确指定将要存入的 token 的数量`amountIn`，以及接受的输出 token 的最低数量`amountOutMin`。
- `swapTokensForExactTokens`：意味着接收的输出 token 的数量是固定的。用户需要准确指定想要接收的 token 的数量`amountOut`，以及需要存入的 token 的最大数量`amountInMax`。
- `path`：用户可以指定需要交换的 token 路径，可以跨越多个池子。
- `to`：接收地址
- `deadline`:最晚时间

### 代码解析

```solidity
amounts = UniswapV2Library.getAmountsOut(factory, amountIn, path);
require(amounts[amounts.length - 1] >= amountOutMin, 'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT');
```

当调用`swapExactTokensForTokens`时，首先根据参数计算单个 swap 或跨越多个池子的 swap 预期的输出，如果预期的输出低于用户指定的最低输出，函数将 revert。对于`swapTokensForExactTokens`，计算所需的输入，如果高于用户指定的最大输入数量，则 revert。

```solidity
TransferHelper.safeTransferFrom(
    path[0], msg.sender, UniswapV2Library.pairFor(factory, path[0], path[1]), amounts[0]
);
_swap(amounts, path, to);
```

- 然后这两个函数都会将用户的 token 转入到`path`中第一个池子中去。这里采用先转账、再调用无回调`swap`的方式，转账与调用在同一笔交易中原子完成。Pair 本身也支持在 flash swap 回调中支付输入，按最终余额结算。
- 最后它们都调用了`_swap`函数。

<img src="images/Uniswap10.jpg" alt="uniswapV2 _swap代码解析" width="50%" height="50%">

上图中，若为最后一跳，输出转给用户指定的`to`；否则直接转给下一个 Pair。

## \_addLiquidity

在 [mintAndburn 章节](UniswapV2-mintAndburn.md#流动性比例检查)，我们提到过流动性安全检查。具体来说，我们希望确保存入的两种 token 数量与池子中的 token 余额比率是相同的，否则，我们获得的 LP Token 数量为两侧存入量与原储备量比率的较小值，乘以 LP Token 总供应量（`_mintFee`处理后、本次增发前），并向下取整。但是在 LP 发起添加流动性交易和交易被确认之间，池子中资产会发生变化。router 中提供了`_addLiquidity`函数可以为 LP 提供必要的流动性安全检查。

<img src="images/Uniswap11.jpg" alt="uniswapV2 addLiquidity代码解析" width="50%" height="50%">

## removeLiquidity

移除流动性会燃烧 LP Token，如果 token 比率在 LP 发起移除流动性交易和交易被确认之间发生剧烈变化，那么 LP 将无法取回他们预期的 token 数量。router 中提供了`removeLiquidity`函数可以为 LP 提供必要的安全检查。

<img src="images/Uniswap12.jpg" alt="uniswapV2 removeLiquidity代码解析" width="50%" height="50%">

上图中的`amountAMin`和`amountBMin`是用户可接受的最少取出数量。

## 对`Fee on Transfer Tokens`的支持

在 UniswapV2 中，处理`Fee on Transfer Tokens`时需要进行特殊处理，因为这些 token 在每次转账的时候会扣除一部分费用，实际到账量可能小于转账量。

- **swap**：Router02 中名称带`SupportingFeeOnTransferTokens`的 swap 入口只支持固定输入（exact-input），逐跳按 Pair 实际到账量计算输出。
- **removeLiquidity**：`removeLiquidityETHSupportingFeeOnTransferTokens`及其 permit 版本将 Router 持有的该 token 余额转给接收方，但`amountTokenMin`不保证接收方扣费后的净到账量。
- **addLiquidity**：Pair 按实际到账量计算 LP Token，但 Router 的配比和最小数量检查基于转账前金额，不能保证扣费后的净到账比例或数量，超额投入也不会自动退回。

这些入口针对转账扣费的情况处理，并不保证兼容任意非标准 token。
