---
layout: post
title: Fee Credits
subtitle: A fee mechanism that lowers costs and increases engagement for users.
# thumbnail-img: /assets/img/fee-credits.png
share-img: /assets/img/path.jpg
tags: [fee mechanisms, protocol design]
author: Dev Ojha and Alpin Yukseloglu
---

In this article, we propose a "Fee Credit" system, which would allow users to earn non-transferable credits through various actions that benefit the network. These credits would then be used to pay for transaction fees, effectively reducing or eliminating transaction-related costs for many users.

We believe that fee credits have the potential to unlock entirely new user flows and can serve as a powerful lever for protocol designers and app developers to incentivize beneficial user behavior without compromising any of the security properties of their fee market.

## Background

Transaction fees play a crucial role in providing sybil resistance for blockchain protocols. More recently, they have also been treated as a [form of protocol revenue](https://ultrasound.money/), especially with the introduction of fee mechanisms that involve token burns such as [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559). These two roles (sybil resistance and revenue) are often conflated, almost to the point of losing clarity which should be prioritized. Thus, it is important to decouple these two properties of fees to ensure that we are standing on solid ground when discussing fee-related mechanisms.

To be clear: the primary goal of fees is to ensure the security and usability of the network.

They may serve as a form of revenue as a byproduct of this, in the same way that any economic cost is a form of “revenue” for some other party in the system. However, the fundamental purpose of fees is not profit. It is to protect the network from spam.

In this post, we outline below a new mechanism that leans on this fact to allow users to pay less in fees while maintaining security properties of the underlying fee market. The primitive we introduce, however, has much more far-reaching implications than merely lower costs for users.

## Motivation

For a fee mechanism to be sybil-resistant, there must be some scaling economic cost to generating spam. Transaction fees are one way of achieving this, but they are by no means the only way. In fact, transaction fees are in some sense a last resort – an option that trades off important usability and accessibility properties presumably because there don't seem to be any other ways to enforce an economic cost to spam other than literally charging people for it.

But there are. In fact, any chain that has significant economic activity, by definition, has users incurring economic costs for their activities. In such environments, the naive approach of charging the same structure of fees on every transaction (regardless of the other economic costs it generates) is incredibly inefficient.

In other words, the problem with the usual one-size-fits-all fee model is that it does not accommodate the many other forms of credible economic commitments that actors can make in a system.

By introducing a fee credit system, we can bring this broader set of user activities in-scope for our fee markets. Perhaps even more importantly, we can add configurable weights to different activities, nudging users towards desirable actions that contribute to the network while at the same time reducing or eliminating transaction fees for important parts of the user experience.

At a bare minimum, this would drastically lower costs and improve user experience for most users. In the best case, it could unlock an entirely new class of user flows through applications that lean on having low/no fees for users.

## Specification

The design space for fee credits is vast, especially with regards to the behaviors they can be used to incentivize and the user flows they enable. That being said, there are several core properties that are required for a sound implementation of fee credits as a primitive in the way we have described above. To leave the door open for anyone who might want to implement this, we outline below one viable set of such properties, as well as a high level spec for how it might be implemented.

### Fee Credit Properties

A fee credit can be defined as an interface that has the following properties:

- Represented as a non-transferable token (for instance, a native Cosmos SDK coin restricted to be non-transferable)
- Constrained maximum amount of credits per account (excess credits earned would not be granted)
- Usable for transaction fees at a rate of 1 base gas unit = 1 fee credit
- Burned when collected for a transaction fee
- Cannot be used for any form of fee delegation, as would be the case with fee grants implemented by most protocols or, on the Cosmos stack, through Authz. This is to prevent delegated usage of fee credits, which would violate non-transferability.

Each of these properties enforce important constraints to ensure that fee credits serve the purpose they are intended to. While further properties could potentially be added for mechanisms that build off of fee credits as a primitive (e.g. if one wants to create a market for credits), the ones outlined above are kept as minimal as possible so as to not overconstrain the design space.

### Earning Fee Credits

This is the most open-ended component of this fee credit mechanism – there are countless viable options for what might constitute a trigger for fee credits to be earned.

The common thread is simple: **it must introduce a credible economic cost to spamming the system.** This can be as straightforward as counting literal fees that are incurred at the app layer (e.g. swap fees) or something slightly more complex such as distributing credits to stakers due to the time cost of bonding requirements.

As long as this core property is satisfied (and the specific amounts are well parameterized), almost any form of economic activity that results in a net cost to the user can potentially be used as an opportunity to generate fee credits.

With that said, here are a few examples of what we believe are sound, broadly applicable, and immediately useful examples of ways to generate fee credits:

1. **Token Locking**: Users can lock tokens of economic value, such as LP shares or staking tokens, to earn fee credits. This provides sybil resistance and aligns user incentives with the network's health.
2. **Staking Rewards**: Staking rewards can accumulate fee credits for users. When users claim their staking rewards, they will also receive fee credits up to the maximum balance per account. This approach improves upon EOS's conceptualization by giving stakers direct rights over block space with fee credits.
3. **Accepted Governance Proposals**: Users who submit governance proposals that are accepted by the community will receive fee credits as a reward for their positive contributions to the network.
4. **Swap Fees in Large AMM Pools**: Users who provide liquidity in large AMM pools will earn fee credits based on the swap fees they generate. This encourages users to contribute to the liquidity and overall health of the ecosystem and makes **swaps essentially free for most users** in terms of tx fees!

### Spending Fee Credits

There are generally two ways for users to spend their fee credits: either on the transaction that generates them, or on some other transaction in the future. A basic interpretation of the mechanism outlined so far might assume that only the latter would be supported and that fee credits could only be used for activity that comes after the transaction that generates them. We would like to take a moment to demonstrate how incorporating the former approach instead could unlock quite elegant user flows.

As a guiding example, when a user swaps on Osmosis, they pay two fees: a swap fee and a transaction fee. Since the latter is a cost that is primarily pushed onto the user to prevent spam, it should be possible to have it be partially or fully covered by the swap fee paid in the transaction. The net impact of such a feature would be that the user does not have to double-pay on the overlapping portion of the fees in their swap transaction. A framing of this consistent with our fee credit mechanism would be that the swap fee generates fee credits that are immediately consumed towards the same transaction.

The generalized version of this would be the allow all fee credits to be consumed by the transactions that generate them, and only have excess credits remain in the account afterwards. This flow can allow for many important parts of an application's user flow to cost essentially no transaction fees without requiring the chain to provide subsidies.

### Implementation

While fee credit systems can be built into any chain infrastructure, the full breadth and depth of the design space can only really be explored in an app-specific context where the earning and spending operations for credits can be tailored to the context of the application. As a result, we've outlined a brief implementation-level spec to aid with anyone who is interested in building fee credits on the Cosmos SDK, either as a feature for their own chain or as an open source module for other appchains to integrate.

The following steps outline a high-level implementation plan:

* **Create Non-transferrable coins**: We need to have a new, non-transferrable coin for fee credits. We suggest implementing this by using send-hooks. The send hook implementation would:
  - Block all sends from user account to user account for this coin.
  - Allow module account to module account sends.
  - Allow module to user sends, but the transfer amount gets capped s.t. the user can't exceed the max fee credit allowance.
  - Block all sends from user account to module account, except for the fee handling module account.
* **Update Transaction Fee Handling**: Modify the transaction fee handling logic to accept fee credits as payment and burn the credits when used. Furthermore, disallow fee credit usage in a tx that contains any Authz execution.
* **Implement Earning Mechanisms**: Add the earning mechanisms for fee credits to the relevant modules, such as staking, governance, and AMM pools. Initial implementation could just be:
    * x/mint: Sending some number of fee credits per day to the address that distributes staking rewards
    * posthandler: Give some number of fee credits directly upon successful tx with sufficient amount staked

## Open Question: Inclusion Guarantees

While the fee credit mechanism described in this article has many benefits, it does face some headwinds in specific applications that do not involve any direct compensation/tip for validators. Specifically, in cases where we expect fee credits to cover the entire transaction fee, we run the risk of validators having no incentive to prioritize the transaction over ones that do pay some priority fee or tip. In cases where nothing at all is paid to the validator, we also run the risk of validators censoring the transaction algother because there is no direct incentive for taking on the compute cost of inclusion.

We have a number of ideas on how this could potentially be remedied (ranging from changing tip distribution logic to earmarking chain-enforced or chain-subsidized transaction lanes), but we are leaving this point here as an open research question instead as we believe that this is an interesting frontier with many viable and diverse potential paths forward.

It is important that the long term inclusion problem is solved if we want to ensure that fee-free transactions can gain adoption without leaning on goodwill of validators. That being said, for many transaction types (such as swaps with taker fees distributed to stakers), there will be an inherent incentive to include the transactions so this will be less of any issue. In the slice of cases in which no fees flow to validators, however, it is important to consider the implications of using a fee credit system on long term inclusion guarantees so that one can ensure the system functions as intended.

## Conclusion

Fee credits can serve as a powerful tool for protocol designers and app developers to provide low-fee or even fee-free user flows for their applications, paving the way for a new class of more accessible, cost-effective, and compelling user flows.

If you are interested in working on this or have ideas for how the mechanism can be improved/built on, please reach out on Twitter/X @0xalpo or @valardragon!