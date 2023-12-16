---
layout: post
title: The Dangers of Portfolio Margining
subtitle: How a seemingly simple UX improvement can lead to unbounded fragility to build up in derivatives products.
# thumbnail-img: /assets/img/fee-credits.png
share-img: /assets/img/portfolio-margining.png
tags: [perps, protocol design]
author: Alpin Yukseloglu and Dev Ojha
---

Our industry has always had a particularly high appetite for leverage. Leverage-related products have long been the dominant driver of volume across all cryptoassets, and any new feature that has enabled users to safely take on more exposure for less capital has, for the most part, spread fast and far to most major exchanges and assets.

In this post, we would like to explore the implications of one such feature that a [growing](https://support.kraken.com/hc/en-us/articles/4975925630484-Futures-Trading-Glossary) [number](https://www.binance.com/en/blog/futures/what-is-the-available-balance-margin-balance-and-total-balance-on-binance-futures-457299340443288694) of [derivatives](https://help.dydx.exchange/en/articles/4797405-how-does-cross-margining-work-on-perpetuals) [exchanges](https://news.opnx.com/25248-usdt-perp-positions-migration-to-ousd-portfolio-margining) have implemented: **portfolio margining**.

## Overview

To ensure solvency, derivatives products count unrealized losses towards an account's margin. Portfolio margining makes the simple contribution of counting unrealized profits towards it as well.

In general, this feature is marketed as merely a UX improvement: your profits automatically offset your losses and you can compound growing positions without needing to take profits.

But portfolio margining has a deeper implication that critically impacts the nature of the risk in any system that implements it: **it removes the distinction between realized and unrealized value**.

The operative component of this feature when it comes to the risk engine simple: it allows users to re-lever on unrealized profits... without ever closing a position!

It is clear that such a feature creates space for new leverage to be taken. But just how much leverage does portfolio margining enable? In this post, we explore the answer to this question.

## Hidden Non-linearities

At a first glance, one might assume that the new capacity for leverage from portfolio margining would be something in the ballpark of adding a new collateral type or decreasing margin requirements.

We demonstrate below that this assumption is surprisingly (and exceptionally) wrong. In general, this feature seems to allow for tremendous amounts of leverage to build up in very poorly understood ways.

Perhaps the greatest cause for concern is that this buildup is demonstrably (and somewhat unintuitively) amplified by multiple layers of non-linear behavior. Thus, it seems as though otherwise small changes (such as increasing the allowed leverage factor from 5x to 6x) can increase the system's potential leverage far beyond what might be intended or safe.

To help shed light on the shape of this boundary, we show two important properties of letting users lever on unrealized profits:
1. It allows for portfolio value to grow superlinearly and unboundedly, _regardless of sell-side liquidity_
2. This growth scales exponentially with the maximum allowed per-position leverage

The latter is particularly dangerous because it can fool even those who are well aware of the former. Any derivatives exchange that does not have a strong grasp on both of these properties and how they relate to/amplify each other is at risk of wildly misparameterizing their protocol and exposing their markets to significant tail risk.

Ultimately, our goal is to highlight the fact that this frequently glossed over feature can lead to surprising amounts of fragility building up in systems that implement it.

An implicit secondary goal is to hint at the fact that this feature might perhaps introduce a bit too much complexity for retail traders to safely interact with -- its already complex for system designs to reason about!

## Core Properties

As a guiding example, we isolate a specific limit case of a portfolio that leans heavily on portfolio margining: a long position that consistently re-levers on an asset that steadily increases in price.

It might seem as though this is an unrealistic (or at least improbable) scenario, but we show below that the risk buildup can be so drastic that even relatively minor increases in price can trigger an intense blowup in leverage (and thus capacity for bad debt).

The general approach outlined below should carry cleanly to shorts as well, but we leave this as an exercise to interested readers as it does not materially impact the claims we make.

### Property 1: Superlinear Exposure Growth (Regardless of Sell-Side Liquidity)

Traditional leverage allows traders to increase their exposure by a constant factor against an underlying asset. In other words a 3x levered position would roughly gain 3x the exposure to both the upside and the downside (adjusted for funding rates, interest rates, fees etc.).

A critical component of such systems is that any position, whether it is levered or not, grows linearly with respect to price growth. In other words, the position's exposure grows in direct proportion to the growth of its underlying asset.

If a trader attempts to re-lever a growing position, they can do so by taking profits and redepositing them to approximate a superlinear payoff curve. This process, however, has an implicit rate limiter in the form of market liquidity: to realize profits, a trader must close their position against other willing traders. This implies that with traditional leverage, any superlinear growth of exposure is subject to liquidity constraints on the exchange and cannot grow unboundedly.

Does portfolio margining maintain this property?

To explore the answer to this question, let’s set up a scenario that covers the limit case with the following assumptions:
- Price increases in uniform ticks (e.g. a 0.1% increase on each tick). These can be thought of as oracle updates in an environment where price steadily increases
- On each tick, the position owner re-levers their position against their unrealized gains

For the sake of example, we take the tick size of price increases to a very small number so we can approximate the behavior of a continuous process.

If we track the asset value of such a construction as price increases, we get the following behavior (assuming a tick size of 0.1% growth, initial collateral of 1 USDC, and leverage factor of 3x):

<img width="690" alt="leverage chart 1" src="https://github.com/AlpinYukseloglu/AlpinYukseloglu.github.io/assets/62043214/37bc4a1e-7f33-4280-8f2e-c503b6eb9f19">

Given that the x-axis shows a linear increase in price, it should be abundantly clear that the portfolio above grows superlinearly with respect to price increase.

The liabilities plot shows us something perhaps even more concerning: for a mere $1 of collateral, a position can take on tens of dollars of debt without ever settling a position.

As we see, both the portfolio asset  and liabilities curves follow power equations, although we leave the proof for this to a future post.

<img width="690" alt="leverage chart 2" src="https://github.com/AlpinYukseloglu/AlpinYukseloglu.github.io/assets/62043214/f4fe3e38-eed6-45c5-9ec0-0f10226610b5">

It is worth keeping in mind that while the margin for the position stays relatively stable during this growth, an increasing portion of it is in unrealized profits that will be torn down in the event of even a small price dip.

For example, in the case of a price run-up of 40%, a price drop of less than 4% would be enough to eat into half of the original collateral. For most traders, this would be enough to trigger a cascade liquidation that takes out the entire portfolio.

In general, portfolio margining seems to make it very easy for traders to mismanage their risk and shoot themselves in the foot. Perhaps even more importantly, it allows exposure to grow superlinearly regardless of market liquidity, meaning that a poorly parameterized system can have a potentially unbounded amount of leverage build up.

### Property 2: Exponential Exposure Relative to Max Leverage

While the above property is tricky to deal with and dangerous if ignored, there is another component of portfolio margined systems that is even more insidious: its relationship with the maximum allowed leverage.

Specifically, the question we ask is as follows: **how does the system's capacity for leverage change as one increases the per-position leverage?**

A common response to this might be that the relationship is linear — that increasing allowed leverage from 2x to 3x has roughly the same impact as increasing it from 3x to 4x. 

Unfortunately, this too would be incorrect. In fact, it would not even be close. Let’s see why.

If we take the same example from the previous section and plot it for different leverage factors on re-levering operations (say, 1x through 5x), we get the following shocking result:

<img width="690" alt="leverage chart 3" src="https://github.com/AlpinYukseloglu/AlpinYukseloglu.github.io/assets/62043214/0ff3c956-0680-4632-aa3d-d2f75813f741">

It turns out that even the relationship between assets in a portfolio and leverage is superlinear. If we extend the x-axis further, the asymptotic behavior becomes even clearer:

<img width="689" alt="leverage chart 4" src="https://github.com/AlpinYukseloglu/AlpinYukseloglu.github.io/assets/62043214/eee7f47e-eef0-4cea-b244-d12c67cfd812">

It seems that the relationship is not just superlinear, but exponential!

In other words, increasing allowed leverage by a fixed amount can be _exceptionally_ deceiving in its potential impact on leverage building up in the system.

## Implications

We’d like to reiterate that the goal of this post is not  to put portfolio margining in a negative light; rather, our goal is to shed some light on how adding this feature affects the topology of leverage in the overall system.

With that said, at least one takeaway should be abundantly clear: **portfolio margining is not merely a UX improvement.** It has large and far-reaching implications on the nature of the risk that can build up in any system that implements it.

Furthermore, it is worth highlighting that this feature has particularly unintuitive behavior compared to other features like it. It is packed with nonlinearities in a system that otherwise operates, for the most part, within linear constraints.

One might even venture to say this feature can have a _negative_ impact on user experience given how unintuitive it is from a risk management standpoint. If the claims we make in this post are intended to be suprising to system designers, then the average trader is likely ill-positioned to understand the implications of their decisions surrounding features like portfolio margining.

Our hope is that the discussion above demonstrates, at least in part, where these nonlinearities lie and roughly how they behave in the limit.

## Guardrails

While risk engines are complex and there is no one-size-fits all solution to ensuring portfolio margining is implemented securely, there are a number of design tweaks that can help minimize the risk of unexpected blowup of leverage. We outline three of the many options below as a starting point.

### Add a discount factor to unrealized profit collateral

One potential path forward is to borrow a guardrail commonly used for volatile collateral types: add a discount factor on how much value they count for. For instance, if the discount factor is 0.75, then $100 of unrealized profits would at most count for $75 of collateral.

While this approach does not change the asymptotic behavior of the system (i.e. it still behaves according to power equations), it can still have an outsized impact on dampening the risk the protocol takes on.

This becomes much clearer when one reframes the discount factor as simply allowing for a lower leverage factor on unrealized profits: in our example above, if the previous allowed leverage was $3$x, then the allowed leverage on unrealized profits would be $0.75 \cdot 3 = 2.25$x.

Since we've established that the payoff curve follows an exponential relationship with the leverage factor, even a small discount factor can reduce tail risk drastically. After all, the exponential relationship works both ways!

### Global and per-account caps on unrealized collateral

The primary failure case introduced by portfolio margining – an unbounded buildup of fragile leverage – has an important property: the unrealized profit collateral quickly eclipses the regular collateral. Thus, if we are able to add a constraint around this property, we should be able to provide stronger guarantees around solvency in tail scenarios.

To implement this, we can introduce a parameter that represents the maximum percentage of global collateral that can come from unrealized profits. We can also go a step further and add a second parameter that does the same on a per-position basis.

These parameters can either be fixed (similar to margin requirements), or they can vary based on factors such as global leverage and liquidity to get more granular behavior in the risk engine. However, in the spirit of not further complicating features that are already likely far too complex for even informed traders, fixed parameter values as simple guardrails seems appropriate.

### Bonded Insurance Fund

One challenge with attempting to factor in market liquidity naively (e.g. checking 2% depth on the book) is that these numbers can be easily manipulated by an attacker. Furthermore, given the reflexivity of liquidity in a market, oftentimes these numbers can rapidly change even when the system is not being attacked.

As an interesting alternative to leaning on normal market liquidity, one can introduce a separate bonded insurance fund that has the sole purpose of backstopping tail scenarios triggered by portfolio margining.

One viable construction for this is to allow users to deposit their funds into a pool that strictly buys assets at a discount to the index price. To provide strong liquidity guarantees to the protocol, this pool would require users to lock/bond their funds for a period of time (either on a fixed timelock starting at deposit time or a bonding-like system where the clock starts at withdrawal time).

Then, in the case of a severe and unexpected liquidity shortage, this pool would be used to buy assets off the books at a fixed discount to the index/external market price. For instance, if BTC index price is $30k and the discount is 10%, then the insurance fund would serve as a backstop to buy BTC at $27k or below until the fund is depleted. The same could be replicated for the BTC sell-side using a second pool.

Efficient matching and execution of these orders can be achieved with an oracle-based order book mechanism where the tick prices shift according to an external oracle price. We will be writing about a generalized version of such a mechanism in a future post, but if anyone would like to learn more in the meantime, please do not hesitate to reach out!

## Future Work

The specific scenarios we analyze above explore a tiny slice of the broader risk landscape related to derivatives products and leverage. Perceptive readers might have noticed parallels with various other concepts and financial primitives in this domain. Although there are many directions one can take this work, we would like to highlight one path that seems particularly interesting and rich in implications.

If one examines the plots demonstrating the effect of increasing the leverage factor, the curves carry an uncanny resemblance to a seemingly completely unrelated product: [power perpetuals](https://www.paradigm.xyz/2021/08/power-perpetuals).

While the relationship is more complex than merely plotting $ETH^L$ on our graphs above, it is clear that some relationship exists here, and more formally deriving what exactly it is (and how it varies with the leverage factor $L$) would be an extremely interesting future direction to explore.

The conclusion one can draw from this, if successful, is quite astonishing: adding the ability to lever on unrealized profits would be equivalent to enabling full-feature power perps! A realization like this might, for example, have serious implications for how funding rates are calculated in such systems (as well as for many other parts of the risk engine as well).

While there are many ways to approach a problem like this, here is a candidate guiding question: what happens if we expand the operation we analyze above to accommodate price decreases by _de-levering_ on the way down?

**If anyone is interested in discussing anything related to the ideas in this article, building on top of the concepts we've covered, or contributing to any future work on this topic, please reach out to us on Twitter/X @0xalpo and @valardragon!**
