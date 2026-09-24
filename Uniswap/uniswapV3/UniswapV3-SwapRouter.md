# Uniswap V3 SwapRouter

## SwapRouter 是什么

在 Swap 章节，我们已经学过 pool 合约的 `swap`函数，看起来参数很少，但直接与其交互技术门槛较高：调用者要明白`amountSpecified`的正负含义，设置`sqrtPriceLimitX96`以防滑点，自己实现回调函数，并在多跳场景下自行拆分与串联多次 swap。因此，`SwapRouter` 的价值就在于把这些复杂性封装起来，为用户提供安全，易用的交易接口。

## exactInput

这个函数的作用是：给定固定数量的输入 token，兑换成输出 token

### 参数结构

```solidity
struct ExactInputParams {
    bytes path;            // 交易路径（编码格式的多跳路径）
    address recipient;     // 最终接收输出 token 的地址
    uint256 deadline;      // 截止时间
    uint256 amountIn;      // 输入 token 数量
    uint256 amountOutMinimum; // 最小可接受输出，防止滑点过大
}
```

`path`是一个压缩过的路径，它的格式如下：

```solidity
tokenA (20 bytes) | fee (3 bytes) | tokenB (20 bytes) | fee (3 bytes) | tokenC (20 bytes)
```

### 代码解析

```solidity
function exactInput(ExactInputParams memory params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountOut)
{
    // 1️⃣ 第一跳由调用者付款
    address payer = msg.sender;

    // 2️⃣ 逐跳兑换（例如 A→B→C）
    while (true) {
        // 是否还有下一跳
        bool hasMultiplePools = params.path.hasMultiplePools();

        // 3️⃣ 执行当前一跳，返回的输出作为下一跳输入
        params.amountIn = exactInputInternal(
            params.amountIn,                                 // 当前输入 token 数量
            hasMultiplePools ? address(this) : params.recipient, // 中间输出留在 Router，最终输出发给指定接收方
            0,                                                // sqrtPriceLimitX96：0 表示使用交易方向对应的默认价格边界
            SwapCallbackData({
                path: params.path.getFirstPool(),             // 仅取 path 的第一段 (tokenIn | fee | tokenOut)
                payer: payer                                 // 本跳付款方
            })
        );

        // 4️⃣ 更新付款方和路径，或返回最终输出
        if (hasMultiplePools) {
            // Router 用已收到的 tokenOut 继续付款
            payer = address(this);

            // 跳过当前 token，更新 path 为下一段 (B→C)
            params.path = params.path.skipToken();
        } else {
            // 最后一跳的输出就是最终输出
            amountOut = params.amountIn;
            break;
        }
    }

    // 5️⃣ 检查最终输出是否达到最低要求
    require(amountOut >= params.amountOutMinimum, 'Too little received');
}

```

## exactOutput

这个函数的作用是：指定固定数量的输出 token，计算并支付所需的输入 token

### 参数结构

```solidity
struct ExactOutputParams {
    bytes path;              // 交易路径（倒序编码的路径）
    address recipient;       // 最终接收 output token 的地址
    uint256 deadline;        // 交易有效期（时间戳）
    uint256 amountOut;       // 想要拿到的 output token 的精确数量
    uint256 amountInMaximum; // 用户愿意支付的 input token 最大数量
}
```

在这里，`path`是从目标 token 开始倒推输入 token，所以编码顺序是从输出到输入

### 代码解析

```solidity
function exactOutput(ExactOutputParams calldata params)
    external
    payable
    override
    checkDeadline(params.deadline)
    returns (uint256 amountIn)
{
    // Step 1: 从正向末跳倒序执行 swap，回调中递归调用上一跳。
    // payer 固定为调用者，只在最深一层回调支付正向首跳的输入。
    exactOutputInternal(
        params.amountOut, // 想拿到的最终输出 token 数量（精确值）
        params.recipient, // 最终接收 token 的地址
        0,                // sqrtPriceLimitX96：0 表示使用交易方向对应的默认价格边界
        SwapCallbackData({
            path: params.path,       // A → B → C 的路径编码为 C | feeBC | B | feeAB | A
            payer: msg.sender        // 用户是最终付款方
        })
    );

    // Step 2: 取回最深一层回调缓存的正向首跳实际输入
    amountIn = amountInCached;

    // Step 3: 检查实际输入是否超过用户允许的最大值
    require(amountIn <= params.amountInMaximum, 'Too much requested');

    // Step 4: 重置缓存
    amountInCached = DEFAULT_AMOUNT_IN_CACHED;
}

```

这里的默认价格边界是 `MIN_SQRT_RATIO + 1` 或 `MAX_SQRT_RATIO - 1`，仍受协议有效价格范围限制。多跳 `exactOutput` 使用默认边界时，`exactOutputInternal` 还会检查实际输出是否等于指定数量，不足则回退。

## uniswapV3SwapCallback

```solidity
function uniswapV3SwapCallback(
    int256 amount0Delta,
    int256 amount1Delta,
    bytes calldata _data
) external override {
    // Router 不支持整个 swap 都发生在零流动性区间；Core 回调允许两个 delta 同为 0
    require(amount0Delta > 0 || amount1Delta > 0);

    // 解码 Router 在 swap 时 encode 的 SwapCallbackData
    SwapCallbackData memory data = abi.decode(_data, (SwapCallbackData));

    // 验证调用者是合法池子，防止恶意调用
    (address tokenIn, address tokenOut, uint24 fee) = data.path.decodeFirstPool();
    CallbackValidation.verifyCallback(factory, tokenIn, tokenOut, fee);

    // 判断当前池子要求支付的代币和数量
    // amount0Delta > 0 表示池子要求支付 token0，amount1Delta > 0 表示要求支付 token1
    (bool isExactInput, uint256 amountToPay) =
        amount0Delta > 0
            ? (tokenIn < tokenOut, uint256(amount0Delta))
            : (tokenOut < tokenIn, uint256(amount1Delta));

    if (isExactInput) {
        // exactInput 直接支付 tokenIn 给当前池子
        pay(tokenIn, data.payer, msg.sender, amountToPay);
        // 第一跳的 data.payer 是调用者，后续跳是 Router
        // msg.sender 是池子地址
    } else {
        // exactOutput 情况：可能是多跳 swap
        if (data.path.hasMultiplePools()) {
            // 倒序路径中还有上一跳池子
            // 1. 跳过当前输出 token，进入正向兑换的上一跳
            data.path = data.path.skipToken();

            // 2. 递归调用 exactOutputInternal
            //    让上一跳的输出直接支付给当前池子（msg.sender）
            // 3. 最深一层回调才从用户钱包支付正向首跳的输入 token
            exactOutputInternal(amountToPay, msg.sender, 0, data);
        } else {
            // 倒序递归终点，即正向首跳：
            // 直接从用户钱包支付给池子
            amountInCached = amountToPay;
            tokenIn = tokenOut; // swap in/out 因为 exactOutput 是倒序计算
            pay(tokenIn, data.payer, msg.sender, amountToPay);
        }
    }
}
```

从这里结合上面两个函数，`Router`扮演了资金调度的角色：

1. exactInput：

- 用户授权 Router 支付第一跳的 tokenIn，然后每次 pool 回调 `uniswapV3SwapCallback` 时，Router 将 token 转给当前池子（pay(tokenIn, payer, pool, amount)）。

- 中间跳的输出 token 暂时保存在 Router 内部（recipient 设置为 Router 地址），

- 最后一跳的接收方由`params.recipient`指定，不一定是付款用户。

2. exactOutput：

- 以 A → B → C 为例：先调用正向末跳的 B/C 池，向接收方发送 C。

- B/C 池回调 Router，Router 再调用 A/B 池，将输出的 B 直接发送给等待结算的 B/C 池。

- 最深一层回调从用户钱包支付 A 给 A/B 池，再逐层返回完成结算，用户只需提供正向首跳的输入 A。
