pendle开发文档：https://api-v2.pendle.finance/core/docs

pendle开发demo：https://github.com/pendle-finance/pendle-examples-public/tree/main

pendle SDK文档：https://docs.pendle.finance/pendle-v2/Developers/Backend/HostedSdk

pendle 订单簿文档介绍：https://docs.pendle.finance/pendle-v2/ProtocolMechanics/LiquidityEngines/OrderBook

swap 实例：

GET https://api-v2.pendle.finance/core/v2/sdk/1/convert?receiver=<RECEIVER_ADDRESS>&slippage=0.01&tokensIn=0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48&tokensOut=0xf99985822fb361117fcf3768d34a6353e6022f5f&amountsIn=1000000000&enableAggregator=true&aggregators=kyberswap&additionalData=impliedApy,effectiveApy

```jsx
export async function swapTokenToPt() {
  const res = await callSDK<ConvertResponse>(`/v2/sdk/${CHAIN_ID}/convert`, {
    tokensIn: `${USDC_ADDRESS}`,
    amountsIn: 1000000000,
    tokensOut: `${PT_ADDRESS}`,
    receiver: RECEIVER_ADDRESS,
    slippage: 0.01,
    enableAggregator: true,
    aggregators: "kyberswap",
    additionalData: "impliedApy,effectiveApy",
  });
}
```

**整个套利过程：**

### **1、检测订单簿内部是否存在套利空间：**

对应api文档：https://api-v2.pendle.finance/core/docs#tag/limit-orders/get/v1/limit-orders/takers/limit-orders

*—LimitOrderType { 0 : TOKEN_FOR_PT, 1 : PT_FOR_TOKEN, 2 : TOKEN_FOR_YT, 3 : YT_FOR_TOKEN }*

*—sortOrder：asc，desc*

**1.1) YT买方挂单(买一 im相对最高)：[https://api-v2.pendle.finance/core/v1/limit-orders/takers/limit-orders?skip=0&limit=10&chainId=42161&yt=0xbf72d17a4be0eeffe1cbab96b5d64392fb1e6bea&type=2&sortBy=Implied Rate&sortOrder=](https://api-v2.pendle.finance/core/v1/limit-orders/takers/limit-orders?skip=0&limit=10&chainId=1&yt=&type=0&sortBy=Implied%20Rate&sortOrder=asc)desc**

— 得出YT最高买价apy以及数量  yt_order_max_buy_apy、yt_order_max_buy_amont

**1.2) YT卖方挂单(买一 im相对最低)：[https://api-v2.pendle.finance/core/v1/limit-orders/takers/limit-orders?skip=0&limit=10&chainId=42161&yt=0xbf72d17a4be0eeffe1cbab96b5d64392fb1e6bea&type=3&sortBy=Implied Rate&sortOrder=](https://api-v2.pendle.finance/core/v1/limit-orders/takers/limit-orders?skip=0&limit=10&chainId=1&yt=&type=0&sortBy=Implied%20Rate&sortOrder=asc)asc**

— 得出YT最低卖价apy以及数量  yt_order_min_sell_apy、yt_order_min_sell_amont

**1.3) PT买方挂单(买一 im相对最低)：[https://api-v2.pendle.finance/core/v1/limit-orders/takers/limit-orders?skip=0&limit=10&chainId=42161&yt=0xbf72d17a4be0eeffe1cbab96b5d64392fb1e6bea&type=0&sortBy=Implied Rate&sortOrder=](https://api-v2.pendle.finance/core/v1/limit-orders/takers/limit-orders?skip=0&limit=10&chainId=1&yt=&type=0&sortBy=Implied%20Rate&sortOrder=asc)asc**

— 得出PT最高买价apy以及数量  pt_order_min_buy_apy、pt_order_min_buy_amont

**1.4)  PT卖方挂单(卖一 im相对最高)：[https://api-v2.pendle.finance/core/v1/limit-orders/takers/limit-orders?skip=0&limit=10&chainId=42161&yt=0xbf72d17a4be0eeffe1cbab96b5d64392fb1e6bea&type=1&sortBy=Implied Rate&sortOrder=](https://api-v2.pendle.finance/core/v1/limit-orders/takers/limit-orders?skip=0&limit=10&chainId=1&yt=&type=0&sortBy=Implied%20Rate&sortOrder=asc)desc**

— 得出PT最低卖价apy以及数量  pt_order_max_sell_apy、pt_order_max_sell_amont

![pendle-YT-PT撮合.jpg](attachment:4ed50601-599b-4363-ad81-37447b76d21b:pendle-YT-PT撮合.jpg)

**以上为YT和PT买单撮合图，使用了闪电贷; YT卖单和PT卖单撮合类似，但没使用闪电贷。**

**判断：**

```jsx
//买YT-> 卖YT
if(yt_order_max_buy_apy - yt_order_min_sell_apy>5%): //**YT**价差达到设定的阈值 5%，套利成立
		// 买入最小金额，防止库存，后续优化：循环查看YT的买二、买三等和卖二、卖三等是否有利润
		买入, 数量：min(yt_order_max_buy_amont, yt_order_min_sell_amont)-> YT限价单最优卖单
		卖给-> YT限价单最优买单  //低im apy买入 高 im apy 卖出(im apy 越高，YT越贵)

else if(pt_order_max_sell_apy-pt_order_min_buy_apy > 5%): //PT买单和 PT卖单之间，套利成立
		//PT限价买入价格 比 卖单 贵，则先买入便宜PT，再高价卖出
		买入, 数量：min(pt_order_max_sell_amont, pt_order_min_buy_amont)-> PT限价单最优卖单
		卖出->PT最佳买单     //高im apy买入 低 im apy 卖出(im apy 越高，PT越便宜)
							
else if(yt_order_max_buy_apy - pt_order_min_buy_apy > 5%)：//YT买单和PT买单之间，套利成立
		//直接从swap买入YT，这时候会撮合到低im apy的PT限价买单
		//买入最小金额，防止库存，后续优化：循环查看YT和PT的买二、买三等是否有利润	
		买入, 数量：min(yt_order_max_buy_amont, pt_order_min_buy_amont)->pt限价单最优买单 
		卖给->yt限价单最优买单     
		
else if(yt_order_min_sell_apy+5% < pt_order_max_sell_apy): //YT卖单和 PT卖单之间，套利成立
		//先swap买入YT的最低卖单，然后swap 卖出，这时候会撮合到最高im apy的PT卖单
		买入, 数量：min(yt_order_min_sell_amont, pt_order_max_sell_amont)->yt限价单最低卖单
		卖出->撮合Pt限价单最优卖单 			
```

### **2、amm和限价单价格差套利策略(二期优化，暂时不考虑开发)**

**获取amm流动性**

对应api文档：[https://api-v2.pendle.finance/core/docs#tag/markets/get/v1/{chainId}/markets/{address}/swap-amount-to-change-implied-apy](https://api-v2.pendle.finance/core/docs#tag/markets/get/v1/%7BchainId%7D/markets/%7Baddress%7D/swap-amount-to-change-implied-apy)

**2.1) 以限价单最高买入价为基准，在最高买入价的基础上-5%（利润率，可配置）是否有amm卖单，有的话则买入后，再卖给限价单最高买入者**

传入 {order_max_buy_apy-5%}

https://api-v2.pendle.finance/core/v1/42161/markets/{address}/swap-amount-to-change-implied-apy?tokenIn={YT}&tokenOut={SY}&targetImpliedApy={order_max_buy_apy-5%}

— 得出amm_order_sell_all_amount, 即AMM+order在[0, {order_max_buy_apy-5%} ]区间内的卖单流动性 

— 由于之前已经判断过order内部不存在套利空间，故这个流动性是有amm提供的，实际上：

amm_order_sell_all_amount = amm_min_sell_amont

**判断：**

```jsx
if order_max_buy_amont > amm_order_sell_all_amount  :   //存在套利空间
		买入,数量：amm_order_sell_all_amount
		卖给->限价单最优买单  //将低价AMM流动性全部吸收
else： //限价单买单不足以消灭AMM最优卖单，只买部分
		买入,数量：order_max_buy_amont
		卖给->限价单最优买单  //将高价限价买单全部消灭
```

**2.2) 以限价单最低卖出价为基准，在最低卖出价的基础上+5%（利润率，可配置）是否有amm/order买单，有的话则从限价单买入，再卖给价高买入者**

传入 {order_min_sell_apy+5%}

https://api-v2.pendle.finance/core/v1/42161/markets/{address}/swap-amount-to-change-implied-apy?tokenIn={SY}&tokenOut={YT}&targetImpliedApy={order_min_sell_apy+5%}

— 得出amm_order_buy_all_amount, 即AMM+order在[ {order_max_buy_apy-5%}, +无穷大 ]区间内的买单流动性 

— 由于之前已经判断过order内部不存在套利空间，故这个流动性是有amm提供的，实际上：

amm_order_buy_all_amount = amm_max_buy_amont

**判断：**
