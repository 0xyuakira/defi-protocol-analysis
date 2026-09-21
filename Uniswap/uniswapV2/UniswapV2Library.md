# UniswapV2Library

> `UniswapV2Library`简化了与`UniswapV2Pair`的一些交互，并被`router`合约大量使用。它包括几个重要的函数，对于在智能合约中集成 UniswapV2 也很方便。

### getAmountOut、getAmountIn 和 quote

<img src="images/Uniswap14.jpg" alt="uniswapV2 getAmount源码" width="50%" height="50%">

给定输入 token 的数量，可以通过`getAmountOut`计算输出数量，红框中的代码实现了在`swap`章节推导的含费公式。`getAmountIn`则根据指定的输出数量计算所需输入。

`getAmountOut`向下取整；`getAmountIn`使用整数商再加 1，即使恰好整除也会多加 1，因此两者不是整数运算下严格的互逆函数。两侧储备必须非零，输出量必须小于输出侧储备。

`quote`只按储备比例计算另一侧的投入量，即`amountB = amountA * reserveB / reserveA`，供`_addLiquidity`选择配对金额使用。它不包含 swap 的手续费和交易量造成的价格影响，不能代替`getAmountOut`计算交换输出。

### sortTokens 和 pairFor

<img src="images/Uniswap15.jpg" alt="uniswapV2 pairfor源码" width="50%" height="50%">

- `sortTokens`实现了按照给定任意交易对的两种 token 的地址，按照地址大小排序。
- `pairFor`根据工厂合约和两种 token 的地址，按`CREATE2`规则计算 Pair 地址。它只预测地址，不代表 Pair 已部署；其中的`init code hash`必须与目标 Factory 使用的 Pair 创建字节码匹配。

### getReserves

<img src="images/Uniswap16.jpg" alt="uniswapV2 getReserves源码" width="50%" height="50%">

`getReserves`调用了`sortTokens`和`pairFor`方法，并包装 pair 合约的查询储备金方法，实现了给定任意交易对的两种 token 地址，可以查询出这个交易对的储备金。返回的`reserveA`、`reserveB`按传入的`tokenA`、`tokenB`顺序排列。

### getAmountsOut 和 getAmountsIn

<img src="images/Uniswap17.jpg" alt="uniswapV2 getAmounts源码" width="50%" height="50%">

- `path`参数接收一个 token 地址的数组，类似于`[a,b,c,d]`。用户可以转入 a，最终得到 d，这种 swap 跨越了多个池子。
- 通过调用上面的方法，可以查询出(a,b),(b,c),(c,d)这三个池子的储备金额，并依次算出每种 token（包含输入 token）的数量，保存到数组并返回，以便下一步依次调用 UniswapV2Pair 中的 swap 方法。
- 该 Library 和 Router 不搜索最优路径，`path`由调用者提供，通常在链下选择。
