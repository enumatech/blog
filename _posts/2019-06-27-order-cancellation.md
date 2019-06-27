---
layout: post
title:  Order Cancellation in Layer 2 Exchange
date:   2019-07-03 00:00:00 +0800
categories: update
author: Mathis Antony
---

At the time of writing our partner [OAX Foundation](https://www.oax.org) is
about to release our trustless layer 2 exchange on the Rinkeby network. This
version is fully trustless with the exception of the order cancellation
mechanism. In this blog post we will elaborate on the challenges of order
cancellation in both trustful and trustless setting.

<!--more-->

## On-Chain Settlement

If settlement is done on the blockchain (as for example with the 0x protocol)
order cancellation can be implemented by letting clients publish a cancellation
note on the blockchain. The settlement contract can then check if an order has
been cancelled before settling. This may still be subject to
[front-running](https://www.investopedia.com/terms/f/frontrunning.asp) issues.
But because the blockchain is the only source of truth, this solution works and
is rather straight forward.

## Off-Chain

At the heart of the issue with trustless order cancellation lies the fact that
it is not possible to distinguish between a failed delivery of a message (due to
connection problems, for example) and a recipient refusing to acknowledge the
receipt of the message. Without a trusted third party, it is neither possible to
prove that the sender had sent the message nor that the receiver had received
it.

For order cancellation this means that if we are not to rely on the blockchain
as a "trusted third party" to publish the cancellation message, a client will
not be able to cancel orders unless the exchange agrees to the cancellation.
Conversely, for completely trustless cancellation we would have to rely on a
third party. Relying on the blockchain to publish cancellation messages would be
costly, slow and not scalable. This would severely impact the exchange
functionality because order cancellations occur frequently during trading
activities.

Our current order cancellation mechanism however has another weak point: after
the exchange acknowledges a cancellation, it could still fill the cancelled
order until the order expires. This would allow a malicious exchange operator to
fill cancelled orders if the conditions are beneficial, for instance, if there
was a significant price movement since the cancellation. This particular issue
can be fixed without having to fall back to the blockchain for every order
cancellation.

## Non-reversible Off-Chain Cancellation

One way to fix the aforementioned problem is to have the exchange provide a
signed cancellation note that includes information about how much of the
cancelled order has already been filled. The client can then verify this
information and consider the order cancelled if it is correct.

![](/images/order-cancellation/trusless-cancellation.png){: .center-image }

Our layer 2 exchange protocol is described in detail in [our
paper](https://github.com/OAXFoundation/l2x-trustless-exchange/blob/master/docs/l2x-specification.pdf).
In a nutshell, the trustlessness of our protocol relies on a dispute mechanism.
In case of disagreement between the client and the server, the client can open a
dispute in the smart contract. The exchange operator then has to close the
dispute within a certain time frame by providing evidence (including balance
proof, orders and fills) to support that the client's balance is correct. An
honest exchange operator can always close disputes, but a dishonest operator
cannot. Failure to close disputes will lead to the system being halted. At that
point clients can reclaim their funds via the smart contract.

As part of the dispute process the clients and operator provide evidence of
order approvals from the clients and trade execution by the exchange. For
trustless order cancellation we would include the cancellation messages in the
evidence that the client submits. The smart contract would consider this
information when it determines if the trades executed by the exchange were 1.
approved by the user and, 2. not previously cancelled by the exchange. As a
result a client that is in possession of a cancellation message can consider
their order cancelled. If the exchange were to fill it, the client could seek
recourse with the smart contract.

## Non-Cooperative On-Chain Fallback

In order to implement fully trustless order cancellation it is tempting to
combine the two solutions mentioned before. The client could then

1. Try to use the off-chain cancellation mechanism
2. If for whatever reason the previous cancellation did not succeed, fall back
   to the on-chain cancellation mechanism.

In order to implement the on-chain cancellation mechanism we would however have
to make significant changes to the protocol. By design, the smart contract is
not aware of the ongoing trading activity. This is precisely what allows us to
perform the trading activity off-chain without the throughput limitations of the
blockchain. However it also means that when a client registers a cancellation
on-chain their order may have already been filled off-chain. Conversely a
malicious exchange operator could claim the fills happened before the
cancellation occurred even if they did not. We could use a similar mechanism
that we use for disputes where after an on chain cancellation the operator has a
certain amount of time to "cancel the cancellation" by filling the order (at
least partially).

The two cancellation mechanisms are illustrated below.

![](/images/order-cancellation/trustless-cancellation-on-chain.png){: .center-image }

After processing these fills the smart contract could - as part of the dispute
process - enforce that no other fills were created for the order. With this
on-chain cancellation mechanism a malicious exchange still has some time to fill
the cancelled order. If this time window ends before the point in time when the
order expires, the on-chain cancellation mechanism reduces the risk of the
client. In practice however this looks like a rather complex addition to the
protocol so it remains to be seen if the reduction in risk outweighs the cost of
increasing the complexity of the protocol.

I would like to thank my colleagues David, Philippe, Lio and Antoine as helpful
discussions and suggestions for this blog post.

## Author

[Mathis Antony](http://github.com/sveitser/)
