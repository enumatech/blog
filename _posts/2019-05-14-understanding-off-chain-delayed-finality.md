---
layout: post
title:  Understanding Delayed Finality in Off-Chain Transactions
date:   2019-05-14 15:55:00 +0800
categories: update
author: David Leung, Philippe Camacho
---

In an era of instant payments, blockchain transactions seem like a steep step 
back in usability in many ways. Unlike traditional payment methods, blockchain 
transactions are not immediately finalized. This delay in payment completion, 
known as *delayed finality,* is an important concept weaved into the very 
foundation of the blockchain that deserves closer examination.

<!--more-->

In this post, we will provide a brief introduction to delayed finality, and see 
how blockchain transactions, through off-chain solutions, can attain the same 
level of usability as traditional payment, and improve upon it.

## What is delayed finality?

In finance, payment finality refers to the point in time when a payment cannot 
be reversed by the sender or any intermediaries involved. **Delayed finality** 
happens when a payment takes an extended period of time after it was initiated 
to reach finality.

Delayed finality happens for a number of reasons:

| Cause                   | Examples                                                                        |
| ------------------------| ------------------------------------------------------------------------------- |
| Conditional Payments    | Credit card chargebacks, cool-off period required by law, money-back guarantees |
| Asynchronous Processing | Check clearing, layer 2 off-chain solutions                                     |
| Distributed Processing  | Blockchain transactions                                                         |

<br>

## Delayed Finality in Blockchain

Public blockchains are distributed systems where decisions between different 
participants (“nodes”) are reached by following consensus protocols.

As described in Alexis Gauba's [article on blockchain 
finality](https://medium.com/mechanism-labs/finality-in-blockchain-consensus-d1f
83c120a9a), Proof-of-Work (PoW) blockchains such as Bitcoin have "probabilistic 
finality" where, as each new block is appended to the main chain, the 
probability of having the transaction finalized increases. This is why some 
delay is required before the transaction can be safely considered part of the 
blockchain ledger.

Parties interested in the finality of a transaction should therefore wait for a 
number of additional blocks to be mined on top of the transaction block to be 
sure that the payment is finalized. These blocks serves as confirmations, and 
exchanges working with Ethereum typically require 30 confirmations. 

The distributed nature of the blockchain results in delayed finality of the 
transactions.

## Delayed Finality in Off-chain Solutions

The low transaction throughput and high transaction cost for micropayments of 
PoW chains drove researchers to explore scalability solutions built on top of 
the blockchain. One of the most common scalability approach is to use existing 
blockchain as a settlement layer (layer 1) to set up an auxiliary payment 
system so that the parties can use that in a layer above it (layer 2) to 
process transactions without having to go through the blockchain.

Two prevailing approaches in layer 2 designs are payment channels and commit 
chains. Let's look at how delayed finality helps enable those solutions.

## Delayed Finality in Payment Channels

Payment channel is a two-party payment system that requires the transacting 
parties to unanimously agree to opening the channel between them on the 
blockchain. The parties can then process as many transactions off-chain in 
layer 2 as needed by jointly authorizing transactions, signing a record that 
states the agreed balances each party should have.

When the parties have completed the necessary transactions and wish to finalize 
those payments on layer 1, one of the parties may begin the withdrawal process 
by submitting the signed record to the blockchain.

<figure>
  <img src="{{site.url}}/images/off-chain_delayed_finality/payment-channel.png" />
  <figcaption>Figure 1: Visualizing channel-based layer 2 payment system. Credit: Dr. Patrick McCorry. Twitter: @paddykcl</figcaption>
</figure>

To ensure that the record submitted is indeed the latest version, the payment 
channel enters into a state where all the transacting parties are allowed to 
submit their version of the state record during a fixed period. If the version 
on-chain is not the latest version, it will be replaced by the newer one. Once 
the period is over, the record on-chain is used to finalize each party's final 
balance in layer 1.

The asymmetry between requiring unanimous agreement to establish a channel but 
not to finalize and exit the channel may seem odd. This, however, is necessary 
to ensure that the channel can operate trustlessly.

If we required both parties to authorize a channel exit, a party who regrets 
having authorized a transaction could prevent the transaction from being 
finalized simply by refusing to authorize the  exit.

Delayed finality is a key factor in enabling payment channels to operate 
trustlessly.

## Delayed Finality in Commit Chains

While payment channels are a fine solution for two parties to transact 
off-chain, there are some limitations that prevent them from being used in a 
large scale payment hubs:

1. High collateral requirements for a hub operator
2. Long channel setup time
3. Collateral re-balancing
4. Establishing a payment path between two parties without an existing channel 
is non-trivial

As mentioned in an earlier [blog post](https://blog.enuma.io/update/2019/03/08/trustless-noncustodial-exchange.html),
we explored using payment channel to develop a scalable decentralized 
exchange. Upon realizing that we cannot solve the collateral requirement 
problems without reintroducing trust into the system, we looked to other 
approaches and found a solution in commit chains. If you're interested in a 
high level overview of the mechanics of commit chains, please take a look at 
the [Merkle tree](https://blog.enuma.io/update/2019/03/08/trustless-noncustodial-exchange.html#merkle-trees) section of that post.

Here, we are going to focus on explaining how delayed finality helps to solve 
the collateral requirement for a payment hub operator.

With a commit chain, a non-custodial operator is responsible for maintaining 
the ledger that manages the funds deposited into the payment hub by the 
clients, coordinating transactions between them, and posting the cryptographic 
commitments of the balances to the blockchain at regular intervals called 
rounds. The operator takes on the role similar to that of a fund manager, and 
does not require any collateral to operate the commit chain.

Each commit builds on the other. Similar to how additional blocks in the layer 
1 blockchain serves to confirm the blocks before it, a new round added to the 
commit chain confirms the round before it.

<figure>
  <img src="{{site.url}}/images/off-chain_delayed_finality/commit-chain.png" />
  <figcaption>Figure 2: Visualizing commit chain. From our previous blog post, <a href="https://blog.enuma.io/update/2019/03/08/trustless-noncustodial-exchange.html">Trustless, Noncustodial Exchange Prototype</a>.</figcaption>
</figure>

When transactions first get processed by the operator, they are considered 
uncommitted, as they have not been committed to the blockchain yet.

<figure>
  <img src="{{site.url}}/images/off-chain_delayed_finality/uncommitted.png" />
  <figcaption>Figure 3: Off-chain transactions to be added to the commit chain.</figcaption>
</figure>

Once the transactions have been committed by the operator, they become 
disputable. If a client disagrees with the balance that were committed 
on-chain, she can open a dispute by submitting evidence of violation.

<figure>
  <img src="{{site.url}}/images/off-chain_delayed_finality/disputable.png" />
  <figcaption>Figure 4: Once committed, the balances in the updated ledger is subjected to verification and dispute by clients.</figcaption>
</figure>

The commit chain protocol guarantees that an honest operator will be able to 
close any dispute. A dishonest operator, on the other hand, will be unable to 
close this dispute, and will be halted to allow all users to recover their 
funds. Since it is impossible for the operator to perform any actions once it's 
been halted, the transactions from the disputable round cannot be confirmed by 
an additional commit, they never reach finality and are effectively discarded.

<figure>
  <img src="{{site.url}}/images/off-chain_delayed_finality/halt.png" />
  <figcaption>Figure 5: The commit chain is halted if the operator fails to update the balances correctly.</figcaption>
</figure>

On the other hand, if all of the transactions from the previous round withstood 
scrutiny by the clients, the operator will be able to commit new transactions 
from the current round to the blockchain, and by doing so, confirm all the 
transactions from the previous round. At this point, the transactions become 
confirmed and have reached finality.

<figure>
  <img src="{{site.url}}/images/off-chain_delayed_finality/confirmed.png" />
  <figcaption>Figure 6: After another round has been committed, the previously disputable round is now finalized.</figcaption>
</figure>

By introducing delayed finality, the commit chain affords clients the 
opportunity to verify the transactions before they become irreversible. Through 
this validation process, the clients — not the operator — are the 
custodians of their fund.

### Achieving instant finality through operator collateral

Since the operator has no collateral in the fund pool of the payment hub, the 
protocol has to enforce delayed finality to protect client funds from financial 
loss. What if the operator is willing to guarantee their service by depositing 
funds into the hub as collateral?

If the transactions are backed by collateral, they can become finalized the 
moment they are committed on-chain. This is because a client who raised a 
successful dispute can recover their loss against the operator's collateral.

If an operator is only able to deposit enough collateral to only partially 
cover a round's transaction, then those transactions that cannot be covered by 
the collateral will go through delayed finality. The overall system continues 
to be trustless.

## Comparing the different payment systems

| Payment System                | Trustless | Decentralized | Collateralized | Finality |
| ----------------------------- | --------- | ------------- | -------------- | -------- |
| Bank                          | <span style="color: red;">✘</span> | <span style="color: red;">✘</span> | Partially      | Instant<sup>^</sup> |
| Blockchain                    | ✔         | ✔             | Fully          | Delayed  |
| Payment Channel Network       | ✔         | ✔             | Fully          | Delayed  |
| Commit Chain (w/o collateral) | ✔         | <span style="color: red;">✘</span> | Unnecessary    | Delayed  |
| Commit Chain (w/ collateral)  | ✔         | <span style="color: red;">✘</span> | Partially<sup>*</sup> | Instant  |

<small><sup>^</sup> As noted earlier, some services, such as paper checks and credit card 
payments, do not enjoy instant finality.<br/>
<sup>*</sup> A commit chain It could be fully collateralized, though it is not a strict 
requirement
</small>

# The delayed finality trilemma

After examining different approaches, a natural question comes to mind: "when 
or why do we need delayed finality?" One is tempted to propose the following 
trilemma: out of following three desirable properties, only two can be achieved 
together:

- Instant finality
- Trustlessness
- No need for collateral

<figure>
  <img src="{{site.url}}/images/off-chain_delayed_finality/trilemma.png" />
  <figcaption>Figure 7: The delayed finality trilemma: can we build a payment system that is trustless, requires no-collateral and has instant finality?</figcaption>
</figure>

For example there exist trustless payment systems that require no collateral: 
Bitcoin or Ethereum. Bank transfers are instant, yet rely on a trusted party 
and may be (at least partially) collateralized. Permissionless blockchains such 
as Ripple allow near to instant finality w/o collateral, yet they rely on a 
trusted authority in order to validate the identity of the participants. As 
shown in the previous paragraph, commit-chains with collateral enable trustless 
payments with instant finality. By *instant* we mean that once the (signed) 
off-chain transfer message is obtained by the receiver of the payment, there is 
a guarantee that it will be possible to withdraw the corresponding amount. Note 
however that the withdrawal operation is not instant as it involves one or more 
blockchain transactions.

This observation resonates with Vitalik Buterin's scalability trilemma [3], 
which suggests it is hard to get these three properties together: scalability, 
security and decentralization. As in the case of scalability, finding a 
solution that is trustless, provides instant finality and requires no 
collateral, is an interesting topic of research. Proving that such solution 
does not exist would also be very interesting.

## Conclusion: it's all a balancing act

The delayed finality trilemma leads us to believe that when it comes to payment 
systems, there is no perfect solutions. It takes careful deliberation to make 
the right trade-offs to help a payment system meet its objectives.

The trustless nature and optional collateral requirement makes commit chain a 
practical choice for building a trustless, noncustodial exchange, such as the 
one we are working on with the OAX Foundation.

## Authors

[David Leung](https://github.com/dhl), [Philippe Camacho](https://scholar.google.com/citations?user=-7sqCHMAAAAJ&hl=en)

## References

1. Gauba, Alexis. Finality in Blockchain Consensus. 
[https://medium.com/mechanism-labs/finality-in-blockchain-consensus-d1f83c120a9a
](https://medium.com/mechanism-labs/finality-in-blockchain-consensus-d1f83c120a9
a)
2. Antony, Mathis. Trustless, Noncustodial Exchange Prototype. 
[https://blog.enuma.io/update/2019/03/08/trustless-noncustodial-exchange.html](h
ttps://blog.enuma.io/update/2019/03/08/trustless-noncustodial-exchange.html)
3. Ethereum Wiki Sharding FAQ. 
[https://github.com/ethereum/wiki/wiki/Sharding-FAQ#this-sounds-like-theres-some
-kind-of-scalability-trilemma-at-play-what-is-this-trilemma-and-can-we-break-thr
ough-it](https://github.com/ethereum/wiki/wiki/Sharding-FAQ#this-sounds-like-the
res-some-kind-of-scalability-trilemma-at-play-what-is-this-trilemma-and-can-we-b
reak-through-it)
