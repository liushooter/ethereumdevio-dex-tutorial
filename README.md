# ethereumdevio-dex-tutorial

今天听说[1inch](https://1inch.exchange/ "1inch") 发 coin 了，不知道各位老铁有没有领到。有的人暗中窃喜，有人还不了解 1inch，这篇文件就介绍了 1inch 的核心功能。

原文来自 ethereumdev.io 的 [part-1](https://ethereumdev.io/trading-and-arbitrage-on-ethereum-dex-get-the-rates-part-1/ "part-1")，[part-2](https://ethereumdev.io/swap-tokens-with-1inch-exchange-in-javascript-dex-and-arbitrage-part-2/ "part-2")，我们把 2 篇文章合二为一，在内容上做了压缩，文章的主要步骤如下：

- 获得最大的收益兑换方案
- 授权1inch合约操作你的代币
- 利用第一步获得的兑换方案进行交易

## 什么是去中心化交易所聚合器?

去中心化交易所聚合器，即 DEX，以下都用 DEX 表示。
DEX 聚合器是一个平台，它将搜索一组 DEX，以寻找在给定时间和数量下执行交易的最佳价格。

## 1inch DEX 聚合器

1inch 的一大特色就是聚合交易，它会在很多个 DEX 找到收益最大的成交方式。比如 100000dai 想买 x 个 eth，在 uniswap 成交 77%， 在 Bancor 成交 23% ，是最合算的，买到的 eth 最多。

1inch 是由 Anton Bukov 和 Sergej Kunz 开发的 DEX 聚合器，通过一次交易将订单在多个 DEX 之间拆分，给用户提供最好的兑换汇率。1inch 的智能合约是开源的。

[img]

在 1inch 执行交易，过程其实很简单：

- 根据输入的 token 或 ETH 数量，获得预期可兑换的 token 数量
- 授权（Approve）交易所使用你的 token
- 使用第一步的获取的 token 数量进行交易

我们首先仔细了解一下 1inch 的智能合约，让我们感兴趣的是这两个方法：

- `getExpectedReturn()`
- `swap()`

## getExpectedReturn - 估算最佳兑换方案

`getExpectedReturn` 可以随意调用，不需要消耗任何 gas。

这个函数需要传入兑换参数，返回兑换的期望结果，以及交易在各个 dex 之间的兑换比例。

```js
function getExpectedReturn(
    IERC20 fromToken,
    IERC20 toToken,
    uint256 amount,
    uint256 parts,
    uint256 disableFlags
) public view
returns(
    uint256 returnAmount,
    uint256[] memory distribution
);
```

这个方法接收 5 个参数：

- fromToken： 当前拥有的 token 的地址
- toToken： 要交换的 token 的地址
- amount： 想要交换的 token 数量
- parts： 卖出数量拆分成多少份进行最优分布的估算。查看`distribution` 可以了解更多细节，默认是 100
- disableFlags：标记位，用于调整 1inch 的算法，例如可设置禁用某个特定的 DEX

这个方法有 2 个返回值：

- returnAmount：执行交易后将收到的 token 数量。
- distribution：一个 uint256 类型的数组，代表交易在不同 DEX 中的分布情况。 例如，parts 设置为 100，成交额度的 25％在 Kyber 的，成交额度的 75％在 Uniswap，那么 `distribution` 看起来是这样的：[75, 25, 0, 0, …]。

目前 1inch 支持的交易所和排序（与 distribution 对应）如下：

```js
[
  "Uniswap", "Kyber", "Bancor", "Oasis",
  "CurveCompound", "CurveUsdt", "CurveY",
  "Binance", "Synthetix",
  "UniswapCompound", "UniswapChai", "UniswapAave"
]
```

注意：如果你想交易 Eth 而不是 ERC20 token，fromToken 需要设置为特殊的值 `0x0`或
`0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`。

`getExpectedReturn`函数的返回值非常重要，因为接下来需要利用它来执行实际的链上兑换操作。

## swap - 执行多 DEX 兑换交易

要执行链上 token 兑换交易，就需要使用合约提供的另一个函数`swap`。调用`swap`时，需要传入我们之前从`getExpectedReturn`返回的数据，这个操作需要花费 gas。如果要卖出的是 ERC20 token，那么还需要先授权 1inch 合约可以操作你持有的待卖出 token。`swap`函数的定义如下：

```js
function swap(
    IERC20 fromToken,
    IERC20 toToken,
    uint256 amount,
    uint256 minReturn,
    uint256[] memory distribution,
    uint256 disableFlags
 ) public payable;
```

swap 函数接收 6 个参数：

- fromToken：待卖出 token 的地址
- toToken：待买入 token 的地址
- amount：待卖出 token 的数量
- minReturn：期望得到的待买入 token 的最少数量
- distribution：兑换交易拆分分布数组
- parts：执行估算时的拆分数量，默认值是 100
- disableFlags：标记位，例如可设置禁用某个特定的 DEX

## 开发环境搭建

我们将使用 [ganache-cli 分叉(fork)当前的区块链状态](https://ethereumdev.io/testing-your-smart-contract-with-existing-protocols-ganache-fork/ "ganache-cli 分叉(fork)当前的区块链状态")，并提前在 1 个地址上充值了很多 DAI。在示例中，地址是 `0x78bc49be7bae5e0eec08780c86f0e8278b8b035b`。我们还将 gas limit 设置的非常高，因此在测试过程中不至于出现 out of gas 的问题，也不需要在每次交易前估算 gas。启动命令是：

```
ganache-cli -f https://mainnet.infura.io/v3/[YOUR INFURA KEY] \
-d -i 66 \
--unlock 0x78bc49be7bae5e0eec08780c86f0e8278b8b035b \
-l 8000000
```

## 实战 - 估算最佳兑换方案

分析完了 1inch 的关键方法，我们将进行第一笔兑换交易，代码已在 github 开源：`https://github.com/liushooter/ethereumdevio-dex-tutorial/blob/master/part1/index.js`。

```js
var Web3 = require('web3');
const BigNumber = require('bignumber.js');

const oneSplitABI = require('./abis/onesplit.json');
const onesplitAddress = "0xC586BeF4a0992C495Cf22e1aeEE4E446CECDee0E"; // 1plit contract address on Main net

const fromToken = '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE'; // ETHEREUM
const fromTokenDecimals = 18;

const toToken = '0x6b175474e89094c44da98b954eedeac495271d0f'; // DAI Token
const toTokenDecimals = 18;

const amountToExchange = 1

const web3 = new Web3('http://127.0.0.1:8545');

const onesplitContract = new web3.eth.Contract(oneSplitABI, onesplitAddress); // （1）

const oneSplitDexes = ["Uniswap", "Kyber", "Bancor", // （2）
  "Oasis", "CurveCompound", "CurveUsdt", "CurveY", "Binance", "Synthetix", "UniswapCompound", "UniswapChai", "UniswapAave"
]

onesplitContract.methods.getExpectedReturn(fromToken, toToken, // （3）
  new BigNumber(amountToExchange).shiftedBy(fromTokenDecimals).toString(), 100, 0).call({
  from: '0x9759A6Ac90977b93B58547b4A71c78317f391A28'
}, function(error, result) {
  if (error) {
    console.log(error)
    return;
  }
  console.log("Trade From: " + fromToken)
  console.log("Trade To: " + toToken);
  console.log("Trade Amount: " + amountToExchange);
  console.log(new BigNumber(result.returnAmount).shiftedBy(-fromTokenDecimals).toString());
  console.log("Using Dexes:");
  for (let index = 0; index < result.distribution.length; index++) {
    console.log(oneSplitDexes[index] + ": " + result.distribution[index] + "%");
  }
});
```

（1）加载 ABI 以便实例化 1inch 合约实例

（2）该数组指定要使用的 DEX

（3）调用`getExpectedReturn`函数获取兑换方案

代码执行后返回结果类似下面这样：

[img]

此时卖出 1 个 eth，1inch 可以买到 148.47DAI，而 Coinbase 是 148.12。1inch 给出的最佳兑换方案是通过 Uniswap 兑换 96%，再通过 Bancor 兑换 4%，这样可以得到 148.47 DAI，这样比单独通过 Uniswap 或 Bancor 进行兑换都划算。

注意，**这个价格不能作为智能合约的 Oracle 价格，由于各种各样的错误，DEX 可以提供非常低的价格，因此可能会严重操纵这个价格**。

## 实战 - 执行多 DEX 兑换方案

下面我们使用 1inch 聚合器将 1000 DAI 兑换为 ETH。首先定义一些变量，例如合约地址、ABI 等。

```js
var Web3 = require('web3');
const BigNumber = require('bignumber.js');

const oneSplitABI = require('./abis/onesplit.json');
const onesplitAddress = "0xC586BeF4a0992C495Cf22e1aeEE4E446CECDee0E"; // 1plit contract address on Main net

const erc20ABI = require('./abis/erc20.json');
const daiAddress = "0x6b175474e89094c44da98b954eedeac495271d0f"; // DAI ERC20 contract address on Main net

const fromAddress = "0x4d10ae710Bd8D1C31bd7465c8CBC3add6F279E81";

const fromToken = daiAddress;
const fromTokenDecimals = 18;

const toToken = '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE'; // ETH
const toTokenDecimals = 18;

const amountToExchange = new BigNumber(1000);

const web3 = new Web3(new Web3.providers.HttpProvider('http://127.0.0.1:8545'));

const onesplitContract = new web3.eth.Contract(oneSplitABI, onesplitAddress);
const daiToken = new web3.eth.Contract(erc20ABI, fromToken);
```

同时写一个辅助函数来等待交易确认：

```js
function sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
}

async function waitTransaction(txHash) {
    let tx = null;
    while (tx == null) {
        tx = await web3.eth.getTransactionReceipt(txHash);
        await sleep(2000);
    }
    console.log("Transaction " + txHash + " was mined.");
    return (tx.status);
}
```

我们在之前已经获得了兑换比率，现在把代码变的更可读，定义 1 个`getQuote`函数，返回一个包含所有参数的对象。

```js
async function getQuote(fromToken, toToken, amount, callback) {
    let quote = null;
    try {
        quote = await onesplitContract.methods.getExpectedReturn(fromToken, toToken, amount, 100, 0).call();
    } catch (error) {
        console.log('Impossible to get the quote', error)
    }
    console.log("Trade From: " + fromToken)
    console.log("Trade To: " + toToken);
    console.log("Trade Amount: " + amountToExchange);
    console.log(new BigNumber(quote.returnAmount).shiftedBy(-fromTokenDecimals).toString());
    console.log("Using Dexes:");
    for (let index = 0; index < quote.distribution.length; index++) {
        console.log(oneSplitDexes[index] + ": " + quote.distribution[index] + "%");
    }
    callback(quote);
}
```

一旦我们得到了兑换 token 的比率，接下来需要授权 1inch 可以操作我们持有的 token，ERC20 token 标准不允许在一次交易中向合约发送 token 并触发下一个操作。我们写了一个简单的函数，调用`approval`函数，并使用 `waitTransaction` 等待交易确认。

```js
function approveToken(tokenInstance, receiver, amount, callback) {
    tokenInstance.methods.approve(receiver, amount).send({ from: fromAddress }, async function(error, txHash) {
        if (error) {
            console.log("ERC20 could not be approved", error);
            return;
        }
        console.log("ERC20 token approved to " + receiver);
        const status = await waitTransaction(txHash);
        if (!status) {
            console.log("Approval transaction failed.");
            return;
        }
        callback();
    })
}
```

注意，这里演示的时候 **授权额度远远高于当前实际需要的数量**，这样后续就不需要反复执行这个操作了。

接下来就可以调用 1inch 聚合器的 `swap` 函数了。在下面的代码中，我们在调用`swap`函数执行交易后，等待交易确认，并在交易确认后，显示转出账户的 eth 余额和 dai 余额。

```js
let amountWithDecimals = new BigNumber(amountToExchange).shiftedBy(fromTokenDecimals).toFixed()

getQuote(fromToken, toToken, amountWithDecimals, function(quote) {
    approveToken(daiToken, onesplitAddress, amountWithDecimals, async function() {
        // We get the balance before the swap just for logging purpose
        let ethBalanceBefore = await web3.eth.getBalance(fromAddress);
        let daiBalanceBefore = await daiToken.methods.balanceOf(fromAddress).call();
        onesplitContract.methods.swap(fromToken, toToken, amountWithDecimals, quote.returnAmount, quote.distribution, 0).send({ from: fromAddress, gas: 8000000 }, async function(error, txHash) {
            if (error) {
                console.log("Could not complete the swap", error);
                return;
            }
            const status = await waitTransaction(txHash);
            // We check the final balances after the swap for logging purpose
            let ethBalanceAfter = await web3.eth.getBalance(fromAddress);
            let daiBalanceAfter = await daiToken.methods.balanceOf(fromAddress).call();
            console.log("Final balances:")
            console.log("Change in ETH balance", new BigNumber(ethBalanceAfter).minus(ethBalanceBefore).shiftedBy(-fromTokenDecimals).toFixed(2));
            console.log("Change in DAI balance", new BigNumber(daiBalanceAfter).minus(daiBalanceBefore).shiftedBy(-fromTokenDecimals).toFixed(2));
        });
    });
});
```

最后的执行结果看起来是下面这样的：

[img]

我们用 1000 DAI 换回来 5.85 ETH。

在这个过程中，你可能会遇到的这样一个错误提示：“**VM Exception while processing transaction: revert OneSplit: actual return amount is less than minReturn**”。这表示链上的报价已经更新。如果想避免这种情况发生，你可以在代码中引入一个滑点，根据交易金额，将 minReturn 参数减小 1%或 3%。

## 总结

1inch 提供了出色的链上 DEX 聚合实现，可以在一个交易内利用多个 DEX 实现最优的兑换策略。1inch 的 API 使用也很简单，只需要用 getExpectedReturn 估算兑换方案， 然后使用 swap 执行兑换方案，就可以得到最好的兑换结果。你不必总是用 eth 交易，也可以交换 2 个 ERC20 token，甚至可以用 weth 交易。
