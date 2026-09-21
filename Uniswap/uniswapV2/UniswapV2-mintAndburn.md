# Uniswap V2 mint/burn

## 量化流动性

自动化做市商模型不仅提供自动价格发现的算法，也需要按 LP 的份额分配池内资产。由于协议无需许可，任何人都可能添加或撤出流动性，因此需要先量化池子中的流动性，再计算各个 LP 的份额。

| x    | y    | k     |
| ---- | ---- | ----- |
| 1    | 1    | 1     |
| 10   | 10   | 100   |
| 100  | 100  | 10000 |
| 1000 | 1000 | 1e6   |

可以看到，当两种资产数量按相同比例增长时，k 值按该比例的平方增长，而$\sqrt{k}$按该比例线性增长。因此使用$\sqrt{k}$作为流动性的数学度量。它不等同于实际 share 代币数量：交易手续费可以使$\sqrt{k}$增长，而不增发 share。

## 追踪 LP 的流动性份额

在量化流动性后，需要追踪每个 LP 的流动性份额，也就是池子中资产的股份。协议使用了类似 ERC4626 的份额记账思路，但并未实现该标准。它的工作原理是，存入资产并获得另外一个 token（称为 share，也就是 LP Token）。用户的 share 余额除以总供应量，就是他持有的份额比例。实际上，池子合约本身也实现了这个 share 代币的 ERC20 接口。

- 当有人添加流动性的时候，增发 share 代币给他，提升他的份额，稀释他人的份额，相当于股份增持。
- 当有人退出流动性的时候，销毁他的 share 代币，降低他的份额，并发放给他对应份额的资产，相当于减持套现。

这样不必在每一笔交易后更新所有人的 share 余额，只需要在有人添加或退出流动性时，增发或销毁对应数量的 share 代币。交易手续费留在池内，通过每份 share 对应的资产体现；协议费增发另见[mintFee 章节](./UniswapV2-mintFee.md)。

### 如何计算增发/销毁的 share 代币？

定义：

池子中 share 代币的总供应量为$s$，增发或销毁量为$\Delta s$。以下讨论已有流动性的池子，并忽略整数舍入；$s$取`_mintFee`结算后、用户增发或销毁前的值。

量化的流动性为$\sqrt{k}$，池子中原本的流动性为$\sqrt{xy}$。按池中资产比例添加或退出时，新增或退出的流动性为$\sqrt{\Delta x \Delta y}$。

share 代币和流动性份额成正比，则有：

$$
\frac{\sqrt{\Delta x \Delta y}}{\sqrt{xy}} = \frac{\Delta s}{s}
$$

$$
\frac{\Delta x \Delta y}{xy} = \left(\frac{\Delta s}{s}\right)^2
$$

按池中资产比例添加流动性时，满足$\frac{\Delta x}{x} = \frac{\Delta y}{y}$。Pair 也允许不平衡注资，此时按两侧相对投入量的较小值确定流动性份额。例如，当$\frac{\Delta x}{x} > \frac{\Delta y}{y}$时，忽略整数舍入，有$\frac{\Delta s}{s} = \frac{\Delta y}{y}$。

在$\frac{\Delta x}{x} = \frac{\Delta y}{y}$的情况下，结合上面的式子:

$$
\Delta s = \frac{\Delta x \cdot s }{x} = \frac{\Delta y \cdot s }{y}
$$

## mint 代码解析

<img src="images/Uniswap04.jpg" alt="uniswapV2 mint源码" width="50%" height="50%">

`mint`按两种 token 的`balance - reserve`计算本次到账量，不为转账者记录专属余额。直接调用时，转入底层 token 和调用`mint`需要放在同一笔交易中，否则余额差可能被其他人用于铸造份额或通过`skim`转走。router 的`addLiquidity`会原子完成这些操作，`mint`回退时本次转账也一起回退。

### 初始化流动性问题

```solidity
if (_totalSupply == 0) {
    liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
    _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens;
}
```

首次添加流动性时，UniswapV2 会将`MINIMUM_LIQUIDITY = 1000`个最小单位的 share 代币铸造到零地址，永久锁定对应份额，其余份额分配给首次提供者。share 代币有 18 位小数，因此锁定的是$10^{-15}$个完整 share 代币。

用一个简化例子来看完整的攻击过程。假设没有最低锁定份额，协议费关闭，期间没有其他交易，token 和 share 的数量都以最小单位计：

1. **建立初始份额**：攻击者首先存入两种 token 各 1，获得 1 个 share，持有池中全部份额。
2. **捐赠并同步储备**：攻击者再向池子直接转入两种 token 各 99，并调用`sync`。此时两种储备各为 100，share 总量仍为 1，每个 share 对应的资产量被抬高。
3. **让后续 LP 承担舍入损失**：另一位 LP 存入两种 token 各 199，按比例应获得 1.99 个 share，但整数计算向下取整，只铸造 1 个。此时两种储备各为 299，share 总量为 2，新 LP 投入了超过一半的资产，却只获得一半份额。
4. **退出并获利**：攻击者将自己的 1 个 share 转入池子并调用`burn`，取回两种 token 各$\lfloor 299 / 2 \rfloor = 149$。相比累计投入的各 100，净赚各 49；另一位 LP 最后只能取回各 150，相比投入的各 199，损失各 49。

这里利用的是成功铸造份额时的向下取整损失。如果计算出的 share 数量为 0，`mint`会直接 revert，小额 LP 将无法完成添加流动性。

实际合约永久锁定了 1000 个最小 share 单位，这些份额仍计入总供应量，也会分得捐赠带来的资产增长。如果攻击者将每个最小 share 单位对应的资产价值抬高到$V$，锁定份额对应的资产价值就达到$1000V$，这部分资产无法赎回。因此，攻击者需要承担无法收回的资产投入，攻击成本显著提高，但不能保证完全消除攻击。

### 流动性比例检查

```solidity
liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
```

用户将会按两侧相对投入量的较小值获得流动性份额，超额投入不会获得额外份额，也不会自动退回，以此激励用户按池中资产比例添加流动性，保护其他 LP 的资产。所有实际到账的 token 都会被`_update`纳入储备，因此不平衡注资仍会改变池内价格。router 会按储备比例选择转入数量，减少超额投入造成的损失。我们可以看下下面这个例子。

- 假设池子当前有 100 个 token0 和 1 个 token1，LP Token 的总供应量为 10
- 假设 token0 的总价值为 100 美元，token1 的总价值为 100 美元，池子的总资产为 200 美元
- 如果有人添加了 10token0（10 美元）和 1 个 token1（100 美元），总成本 110 美元
- `amount0 * totalSupply / reserve0 = 10 * 10 / 100 = 1`
- `amount1 * totalSupply / reserve1 = 1 * 10 / 1 = 10`
- 如果取最大值，他将获得 10 个 LP token，意味着他拥有了 LP token 总供应量的 50%
- 但是他的流动性资产只占到了池中所有资产的：110 / 310 = 35.4%，相当于他偷走了其他 LP 的资产

## burn 代码解析

<img src="images/Uniswap05.jpg" alt="uniswapV2 burn源码" width="50%" height="50%">

### 移除的流动性是通过池合约收到的 LP token 数量来衡量的

```solidity
uint liquidity = balanceOf[address(this)];
```

直接与`burn`函数交互，需要在调用前向池合约转入 LP token。这两次调用需要在一个交易内，否则其他人可以燃烧掉你的 LP token 并偷走你的流动性。

### 计算移除的流动性资产

```solidity
uint _totalSupply = totalSupply;
amount0 = liquidity.mul(balance0) / _totalSupply;
amount1 = liquidity.mul(balance1) / _totalSupply;
require(amount0 > 0 && amount1 > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_BURNED');
```

这里按销毁份额占总供应量的比例分配两种资产。使用的是实际余额`balance0`和`balance1`，而非旧储备，因此尚未同步的到账资产也参与按比例分配。`_totalSupply`在`_mintFee`之后读取，包含本次可能增发的协议份额。
