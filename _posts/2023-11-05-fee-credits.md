---
layout: post
title: Fee Credits
subtitle: A fee mechanism that lowers costs and increases engagement for users.
thumbnail-img: /assets/img/fee-credits.png
share-img: /assets/img/path.jpg
tags: [fee, protocol, ideas]
author: Dev Ohja and Alpin Yukseloglu
---

# Fee Credits

Transaction fees play a crucial role in providing sybil resistance for blockchain protocols. In spite of transaction fees consistently being termed "revenue" for protocols (especially ones that have fee burns in place), it is important to keep in mind that the primary goal of these fees should not be to generate profit for the protocol, but rather to ensure the security and usability of the network. In order to achieve this, we propose a "Fee Credit" system, which would allow users to earn non-transferable credits through various actions that benefit the network. These credits can then be used to pay for transaction fees, effectively reducing or eliminating costs for users.

## Motivation

If we begin with the goal of building a spam-resistant system, we can make the claim that we want there to be some scaling economic cost to generating spam. Transaction fees are one way of achieving this but are by no means the only way. In fact, in many ways, transaction fees are a last resort – an option that trades off important user experience properties that we would much rather preserve if possible.

The problem with existing fee models is that they do not accommodate the many other forms of credible economic commitments that actors can make in a system. By introducing a fee credit system, we can bring this broader set of user activities in-scope for our fee markets. Perhaps even more importantly, we can add weights to desirable activities, incentivizing users to contribute to the network, while at the same time reducing or eliminating transaction fees for those users. At a bare minimum, this would drastically lower costs and improve user experience for many users; if executed well, it could give protocols a new lever to boost user engagement and drive positive-sum user activity.

## Specification

The design space for fee credits is vast, especially with regards to the behaviors they can be used to incentivize and the user flows they enable. That being said, there are several core properties that are required for a sound implementation of fee credits as a primitive in the way we have outlined above. To leave the door open for anyone who might want to implement this, we have structured this as a specification for one viable mechanism for fee credits.

### Fee Credit Properies

A fee credit would have the following properties:

- Represented as a non-transferable token (for instance, a native Cosmos SDK coin restricted to be non-transferable)
- A maximum amount of credits per account (excess credits earned would not be granted)
- Usable for transaction fees at a rate of 1 base gas unit = 1 fee credit
- When a fee credit is collected for a transaction fee, it is burned
- Cannot be used for any form of fee delegation, as would be the case with fee grants implemented by most protocols or, on the Cosmos stack, through Authz. This is to prevent delegated usage of fee credits, which would violate non-transferrability.

### Earning Fee Credits

This is the most open-ended component of this idea, and there are countless viable options for what might constitute a trigger for fee credits to be earned.

The common thread is simple: **it must introduce a credible economic cost to spamming the system.**

As long as this property is satisfied (and the specific amounts are well parameterized), almost any form of economic activity that results in a net cost to the user can potentially be used here.

That being said, here are a few examples of what we believe are sound, broadly applicable, and immediately useful examples of ways to earn fee credits:

1. **Token Locking**: Users can lock tokens of economic value, such as LP shares or staking tokens, to earn fee credits. This provides sybil resistance and aligns user incentives with the network's health.
2. **Staking Rewards**: Staking rewards will accumulate fee credits for users. When users claim their staking rewards, they will also receive fee credits up to the maximum balance per account. This approach improves upon EOS's conceptualization by giving stakers direct rights over block space with fee credits.
3. **Accepted Governance Proposals**: Users who submit governance proposals that are accepted by the community will receive fee credits as a reward for their positive contributions to the network.
4. **Swap Fees in Large AMM Pools**: Users who provide liquidity in large AMM pools will earn fee credits based on the swap fees they generate. This encourages users to contribute to the liquidity and overall health of the ecosystem.

### Implementation

While fee credit systems can be built into any chain infrastructure, the full breadth of the design space can only really be explored in an app-specific context. As a result, we've outlined a brief implementation-level spec to aid with anyone who is interested in building fee credits on the Cosmos SDK, either as a feature for their own chain or as an open source module for other appchains to integrate.

The following steps outline a high-level implementation plan:

* **Create Non-transferrable coins**: We need to have a new, non-transferrable coin for fee credits. We suggest implementing this by using send-hooks. The send hook implementation would:
  - Block all sends from user account to user account for this coin.
  - Allow module account to module account sends.
  - Allow module to user sends, but the transfer amount gets capped s.t. the user can't exceed the max fee credit allowance.
  - Block all sends from user account to module account, except for the fee handling module account.
* **Update Transaction Fee Handling**: Modify the transaction fee handling logic to accept fee credits as payment and burn the credits when used. Furthermore, disallow fee credit usage in a tx that contains any Authz execution.
* **Implement Earning Mechanisms**: Add the earning mechanisms for fee credits to the relevant modules, such as staking, governance, and AMM pools. Initial implementation could just be:
    * x/mint: Sending some number of fee credits per day to the address that distributes staking rewards
    * posthandler: Give some number of fee credits directly upon succesful tx with sufficient amount staked

## Conclusion

Transaction fees are one of the most misunderstood and mismarketed parts of the stack. These fees are not revenue; they are a sybil resistance mechanism. If we treat them as such, we can cut down tremendous amounts of deadweight loss in a chain's economy while giving protocol designers and app developers a tool to provide low-fee or even fee-less user flows for their applications, enabling a more cost-effective and accessible experience for all users.
