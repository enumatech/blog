---
layout: post
title:  "Decentralized Application Messaging with Whisper — Part 1"
date:   2018-08-08 11:05:00 +0800
categories: update
author: David Leung
---

This is Part 1 in a series of articles on *Whisper*, Ehtereum's
inter-application communication protocol. In this article, we cover the
principles and use cases of Whisper. We will dive into practical examples in
Part 2, and share some tricks we have learned from using Whisper in a
[decentralized exchange project](https://oax.org/) we have been working on.

<!--more-->

## Messaging: One of the three basic needs of Decentralized Applications

Non-trivial applications often require three kinds of resources to provide
services, namely, *Compute*, *Storage*, and *Messaging*. Created with the grand
vision of building a global decentralized computing platform, Ethereum serves
these basic needs with three pieces of foundational technologies: the EVM
(Ethereum Virtual Machine) provides compute, Swarm handles storage of large
files, and Whisper is the answer to messaging.

![Ethereum Ecosystem](/images/whisper/ethereum-ecosystem.png)

*Credit: Vitalik Buterin*

In a nutshell, Whisper is a peer-to-peer (P2P) messaging protocol for
decentralized applications (Dapps) to provide Dapp developers a simple API to
send and receive messages in almost complete secrecy. This devotion to secrecy
forces the designers of Whisper to make some interesting trade-offs to
sacrifice performance for privacy. As a result, Whisper is more suitable to
certain class of use cases.

## Use Cases for Whisper

### What Whisper is good for

- **Publish-subscribe coordination signaling.** Dapps could collaborate with
one another by implementing the pubsub pattern with Whisper.
- **Secure, untraceable communication.** Whisper is designed from the ground up
to support highly private and secure communication with plausible deniability.

### What Whisper isn't good for

- **Ultra low latency/real-time communication.** Whisper messages may be routed
in a probabilistic way, making it difficult to guarantee latency for
time-sensitive applications.
- **Sending large data chunks.** Whisper is best used for messages less than
64KB in size. For larger messages, another channel designed for content
distribution such as swarm may be a better choice.

## Whisper Fundamentals

Whisper is a configurable messaging protocol that gives dapp developers a lot
of flexibility in controlling the security and privacy parameters of their
messages. To fully take advantage of Whisper, it is necessary to understand, at
least at a high level, how Whisper works.

**A network of equal peers**

The Whisper network is made up systems called *nodes* that are connected in a
decentralized way. To establish this network, a node finds its peers in the
network using the
[ÐΞVp2p](https://github.com/ethereum/wiki/wiki/%C3%90%CE%9EVp2p-Wire-Protocol)
 protocol. ÐΞVp2p is a significant piece of technology in the Ethereum
ecosystem, as it provides the foundation for which the protocols in the
Ethereum stack are built. Although they share the same wire protocol, Whisper
and its sister protocols Ethereum and Swarm do not interoperate. The fact that
Ethereum and Whisper nodes tend to runs on the same implementation is simply a
coincidence driven by convenience.

**Identity-driven communication**

Once a node has been connected to Whisper, a Dapp instance can start receiving
messages by establishing an *identity* in that node. An identity is not
strictly necessary to send messages, although it is needed to establish two-way
communications. This gives raise to interesting use cases and challenges.

Broadly speaking, an identity in Whisper is an *entity* (an individual or a
group) that **consumes** messages. In practice, an identity can be thought of
as a holder of an encryption key. To receive Whisper messages, therefore, it is
necessary to create an encryption key. Both symmetric (AES-256) and asymmetric
(secp256k1) keys are supported for different use cases.

Encryption ensures that only the intended recipient(s) can access the content
of a piece of message. If a node can decrypt a piece of message, then the
message is intended for a recipient using that node.

**Delivering Messages in Darkness**

Whisper touts itself as being able to *communicate in darkness*. This means
that two nodes in a Whisper network can communicate without leaving any
traceable evidence to traffic analyzers and other peers, even if those peers
participated in the message routing. This is achieved by trading performance
for privacy.

To achieve total darkness in communication, two basic criteria must be
satisfied: the content of the message is inaccessible to those who intercepts
the messages, and that communicating nodes cannot be easily identified. The
protection of the messages is achieved by Whisper taking an encryption-first
approach to messaging; it is impossible to send unencrypted messages through
Whisper. To hide the fact that two nodes are talking to each other, routing
information must also be hidden. But how?

**Messages are addressed to no one**

To send a piece of message, the sending Dapp makes an API call to its Whisper
node to encrypt the message using a shared symmetric key or the recipient's
public key and seal the encrypted message inside an *envelope*. Like its real
world counterpart, a Whisper envelope contains metadata to help it get routed
to its final destination and processed.

Unlike real world envelopes, Whisper envelopes **do not contain any information
about its recipient**. A typical Whisper envelope has the following format:

    [ Version, Expiry, TTL, Topic, AESNonce, Data, EnvNonce ]

The only piece of information interesting to an attack would be the content of
the data field, but that holds the encrypted data.

It makes sense as a part of the dark communication strategy that the recipient
of a piece of message cannot be readily identified. Without such a crucial
piece of information, the only possible strategy to have any hope for the
message to reach its intended recipient is to send the message to every node.

**Routing in the blind**

Whisper defeats traffic analysis by having each Whisper message routed to every
node in the network. In this sense, Whisper behaves like the user datagram
protocol (UDP) operating in broadcast mode. Since every node receives a copy of
the same message, it is impossible to tell which recipient the sender is trying
to communicate with.

Although it is difficult to tell which recipient a message is addressed to, a
powerful adversary may still be able to determine if two nodes are
communicating if they are able to control all but the two communicating nodes.
In this scenario, if node A sends a message to node B, the adversary will be
able to discern that communication took place between the pair. Such an attack
can be defeated by introducing noise into the network by having well-behaved
nodes sending junk messages encrypted with a random key into the network.

**Probabilistic message filtering**

In order for a node to decide if a piece of message is intended for an identity
using its service, it needs to be able to decrypt the message. Since the
envelope contains no metadata on the intended recipient, a node must try
**every key** it holds before it can determine whether a piece of message is
sent to its users. Decryption is expensive work! Compounded with the fact that
a node will receive every piece of message of the network, it is infeasible to
decrypt every piece of incoming messages in most practical applications.

This problem is solved by requiring each message to be associated with a
*topic*. An identity registers the topics it's interested in by using its
encryption key to create a message filter on a node. This efficient
probabilistic filter, known as a *[bloom
filter](https://en.wikipedia.org/wiki/Bloom_filter)*, can tell to a very high
degree of certainty (false-positives are possible) if a piece of message
belongs to a topic of interest.

A node will only attempt decryption if a filter signals a possible match.

**Quality of Service Assurance**

With every Whisper message being delivered to every reachable node, it is easy
to jump to the conclusion that Whisper is susceptible to denial of service
(DoS) attacks. An attack can be launched in at least two ways:

1. **Flood Attack:** Repeatedly send messages into the network
2. **Expiry Attack:** Make messages hang around for a long time by setting a
long TTL (Time-to-Live) on the envelope

Flood attack is prevented by the Whisper node and the message filters requiring
the sender to perform
*[proof-of-work](https://en.wikipedia.org/wiki/Proof-of-work_system)* (PoW)
computation and including the result as the `EnvNonce` field in the message
envelope. A node may refuse to accept a piece of message if the PoW is too low.
Message filters may opt for more stringent PoW requirement than the node
requires.

Expiry attack is prevented by a message rating system that takes PoW, message
size, and TTL into account to compute a message rating. The criteria of the
rating is rather simple:

1. Smaller messages have higher ratings
2. Messages with higher PoW have higher ratings
3. Messages with lower TTL have higher ratings

The rating of a piece of message affects both its forwarding priority and how
long it will be stored in the system. It is therefore in the interest of the
sender to make sure the messaging rating is as high as necessary to achieve its
objective. For example,when a node's message pool approaches its memory limit,
the node may clean up messages that it consider to be of low rating. This
rating system limits the impact of a DoS attack.

## Barriers to Wider Adoption

As great as Whisper is, the actual uptake of Whisper is somewhat low. This is
partly due to the design choices of Whisper, and partly because of alternative
messaging option from within the Ethereum ecosystem.

**Key Management**

Whisper currently relies on Whisper nodes holding on to users' secret key,
so that messages intended for the users can be decrypted. This mode of
operation rules out being able to use "zero client" providers such as
[Infura](https://infura.io/), one of the most popular Ethereum infrastructure
services, which operate on the principle that wallet software like
[MetaMask](https://metamask.io/) should be responsible for key management.

This limitation can be mitigated by having Whisper nodes running alongside the
wallet software on the user's device. For example, trading application
developed in [Electron](https://electronjs.org/) can implement Whisper for
order signaling and interoperate with regular Whisper nodes. There are some
exciting developments in this space led by
[Guillaume Ballet](https://github.com/gballet), which lay the foundation for
alternative Whisper implementations that operate on protocols other than TCP
and UDP that ÐΞVp2p requires to talk to regular Whisper nodes backed by
[Go Ethereum](https://ethereum.github.io/go-ethereum/) (Geth) or
[Parity](https://www.parity.io/).

**Disabled by Default**

Currently, Whisper is an opt-in service. To use it, one needs to enable to it
at the command line. As a result, Whisper is not running on the
majority of Ethereum nodes. This conscious decision to have Whisper disabled by
default is a sensible one, however, as Whisper is still an experimental
protocol.

**Whisper's Sibling: Postal Services Over Swarm**

In July 2017, a new messaging protocol called *Postal Services over Swarm*
(PSS) was introduced. On the surface, it looks a lot like Whisper but with
a different set of trade-offs, such as having deterministic routing (trading
privacy for performance). Unlike Whisper, which is included with Geth and
Parity, PSS requires a separate Swarm client to operate in a swarm cluster.

While PSS may not be in direction competition with Whisper, having PSS in the
mix may dilute interest and resources available to further the development of
Whisper. On the other hand, PSS may explore alternative solution to the key
management problem that is affecting Whisper at the moment.

In short, PSS represents both a threat and an opportunity to Whisper. Either
way, we as developers benefit from having more options when it comes to
messaging.

## Current Status & Adoption
Whisper is a young, proof-of-concept protocol, but it has already found its way
into production use. [Status](https://status.im/), an interesting Ethereum
client project, uses Whisper to implement chat, and is probably the biggest
production Whisper user at the moment.

We at Enuma are also making use of Whisper for the
[OAX project](https://www.oax.org/) for order signaling.

The latest version of Whisper is V6. Prior versions of Whisper (V2-V5) have now
reached end-of-life and are unsupported.

## Conclusion

Whisper is a simple, privacy-first, low-level messaging protocol for
decentralized applications built on top of the Ethereum blockchain. The
solution shows a lot of promise, and enjoys being tightly integrated with
prominent Ethereum clients such as Go Ethereum and Parity.

We looked some of the obstacles the young protocol faces, and what the
community is doing to address some them.

With a small but vibrant community behind Whisper, it is a messaging solution
worth looking at for decentralized applications.

## Author

[David Leung](https://github.com/dhl)

## References

1. Whisper Wiki Pages.
[https://github.com/ethereum/wiki/wiki/Whisper-pages](https://github.com/ethereum/wiki/wiki/Whisper-pages)
2. Whisper PoC 2 Protocol Specification.
[https://github.com/ethereum/wiki/wiki/Whisper-PoC-2-Protocol-Spec](https://github.com/ethereum/wiki/wiki/Whisper-PoC-2-Protocol-Spec)
3. Whisper v6 RPC API.
[https://github.com/ethereum/go-ethereum/wiki/Whisper-v5-RPC-API](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API)
4. Bloom Filter.
[https://en.wikipedia.org/wiki/Bloom_filter](https://en.wikipedia.org/wiki/Bloom_filter)
5. Backpressure.
[https://en.wikipedia.org/wiki/Back_pressure](https://en.wikipedia.org/wiki/Back_pressure)
