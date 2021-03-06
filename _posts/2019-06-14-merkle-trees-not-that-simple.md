---
layout: post
title:  Merkle trees&#58; it is not that simple!
date:   2019-06-10 15:55:00 +0800
categories: update
author: Philippe Camacho
---

In a previous post [(Trustless, Noncustodial Exchange Prototype)](https://blog.enuma.io/update/2019/03/08/trustless-noncustodial-exchange.html#merkle-trees), we already mentioned Merkle trees and their use in the
exchange protocol we're building with the [OAX Foundation](https://www.oax.org).
Today we will go deeper into the subject, highlightling how easy it is to introduce security flaws when instantiating this seemingly simple cryptographic primitive.

<!--more-->

## The problem

Let us start with the problem.
In many situations, it is required to prove in a secure way that an element belongs to some set:
For example a user contacting a server may need to prove he is part of a white list before being granted access to some resources.
Blockchain protocols such as Bitcoin require to be able to prove that a transaction belongs to a block.
Timestamping documents can be done by proving the document belongs to a certain round of time (minute, hour, day).
The list goes on.

The question is: how can we test securely and efficiently that an element is in the set?
A naive solution may consist of publishing the whole set so that anyone can check for himself whether an element belongs or not.
In order to be sure the information is authentic, one may rely on digital signatures or simply publish the whole set on a public blockchain
 like Bitcoin or Ethereum.
The drawback of this approach is obvious: as the set can be large, the cost of publishing and verifying becomes prohibitive for most of the applications.

So in order to make this membership protocol more efficient we need two things:
 * A short (hopefully constant size) representation of the set.
 * An efficient and secure procedure to check that an element belongs to the set using only the short representation mentioned above
  and maybe some extra (also of small size) auxiliary information.

The first point can be addressed using collision-resistant hash functions (CRHF).

## Collision-resistant hash functions

A CRHF `$H:\{0,1\}^* \rightarrow \{0,1\}^k$` is a function that takes a string of any length as input
a returns a string of fixed length $k$ as output.
Its purpose is to provide a short representation for any kind of information such that this representation is unique in practice, that is if we have `$H(x)=H(x')$` then `$x=x'$`.
[Obviously](https://en.wikipedia.org/wiki/Pigeonhole_principle) as the input set of the function is much bigger that the output set, there exist some values `$x,x'$`
 such that `$H(x)=H(x')$` yet `$x \neq x'$`.
Thus the security requirement<sup id="a1">[1](#f1)</sup> for this function is that it should be hard to find such pair of values
 `$(x,x')$` also called collision.

<!--
$k$ is called the security parameter as it is a way to estimate how hard it is for an attacker to break the function.
Indeed there is a generic attack on all hash functions, called the birthday paradox attack where More on this?
-->
So CRHF are what we are looking for: they allow to represent a potentially huge amount of data with a small number of bits
  composing the hash or digest.

##  Merkle trees

<figure>
  <img src="{{site.url}}/images/merkle-tree/simple-merkle-tree.png" />
  <figcaption>Figure 1: A merkle tree for the set $\{x_1,x_2,x_3,x_4\}$.
  The values of the set are placed at the leaves (in blue). The hash of the set is obtained at the root (in green).
  Conceptually a Merkle tree can be seen as a CRHF for sets (on the right).</figcaption>
</figure>

So now we need to solve the second problem.
Here again naive approaches such as rehashing the whole set that includes one's element are clearly inefficient.
The Merkle tree enables to address our efficiency requirement.
Let `$H$` be a CRHF and `$S=\{x_1,x_2,...,x_{2^l}\}$` a set of size `$|S|=2^l$` where `$l$` is some (small) constant.
The Merkle tree (see Figure 1) is built bottom up, starting with the leaves where we put the values from left to right.
Then for each internal node `$N$`, the value is obtained recursively by hashing the concatenation of the left (`$L$`) and right (`$R$`) child value.
That is `$\val(N)=  H(\val(L)||\val(R))$`. The value `$r$` we obtain at the root is the short representation of the set and indeed
 the hash value of the set.

Now in order to prove that for example element `$x_3$` belongs to `$S$` represented by `$r$` we need to provide
 the authentication path, that is the sibling nodes of the path from the leaf to the root.
 These nodes combined with the the value `$x_3$` are enough to recompute the root `$r$`.
Indeed the verification procedure `$\verif(x,w,r)$` where `$w$` is the authentication path, will return `$\mathtt{true}$`
 if from `$x$` and `$w$` we can recompute `$r$`.
What is attractive about this verification procedure is that it runs in `$O(\log |S|)$` time, as the authentication path
 contains only a logarithmic number of nodes.
This indeed matters, for example if you want to run this check inside a smart contract.

<figure>
  <img src="{{site.url}}/images/merkle-tree/security-proof.png" />
  <figcaption>Figure 2: If the leaves of the attacker's tree differ from the original tree and yet both roots
  are equal, there must be a collision for $H$.</figcaption>
</figure>

The intuition of why the merkle tree construction is secure can be summarized as follows: if `$H$` is indeed a CRHF then an adversary
  should not be able to find two different sets `$X, X'$`
  such that `$\mkr(S')=r=\mkr(S)$`.
The way we argue this is through a [proof by contraposition](https://en.wikipedia.org/wiki/Proof_by_contrapositive):
*If indeed there is an adversary that finds such two sets, then we can find a collision for `$H$` (which we expect to be impossible)*.
To see that we can compare the two trees (see Figure 2) from top to bottom. At the top both roots are equal.
At the second level either all values are the same or there is some difference .
If there is a difference then indeed we found a collision for `$H$` as `$H(y||y') = H(z,z')$`
 but `$y||y' \neq z||z'$`. If all values are equal then we keep going down and apply the same procedure.
At some point we will find different inputs for `$H$` mapped to the same output (as we know that some of the leaves are different).

This reasoning while correct can be misleading as it does not highlight a key assumption we made (implicitly):
**Both trees have the same structure.**

While this statement seems obvious, it might not always be true as we shall see in the next section.
What is important to remember anyways is that **a Merkle tree can be thought as a CRHF for sets with efficient membership testing**.

## How an attacker can leverage the flexibility of the tree structure

<!--
Simple example where the attacker cannot choose the values
-->

So if you are an attacker what would you try to do?
As the goal of a Merkle tree is to verify that an element belongs to some set $S$, an attacker will
 try to find $x' \notin S$ and convince a verifier of the opposite  statement, that is $x \in S$.
A way to achieve this is to find two different sets `$S, S'$` such that $\mkr(S)=\mkr(S')$.
In practice this means building to different trees $T$ (for $S$) and $T'$ (for $S'$)
that yield the same root. If this is possible then the attacker will be able to provide a convincing
 proof for some element `$x' \in S' \setminus S$` (that is $x'$ belongs to $S'$ but not to $S$) and
 thus fool the verifier.

<figure>
  <img src="{{site.url}}/images/merkle-tree/tree-struct-attack-simple.png" />
  <figcaption>Figure 3: A simple attack on the tree structure. The attacker cannot choose values $x_5$ and $x_6$.</figcaption>
</figure>

Let us have a look at Figure 3 to see how this can be done:
On the left we have a Merkle tree corresponding to the (legitimate) set `$S=\{x_1,x_2,x_3,x_4\}$`.
In order to find a collision the attacker can simply remove the leaves and obtain the smaller tree on the right.
The values at the leaves of this new tree are `$x_5:=H(x_1||x_2)$` and `$x_6:=H(x_3||x_4)$`.
So now we have two different sets `$S=\{x_1,x_2,x_3,x_4\}$` and `$S'=\{x_5,x_6\}$` such that
`$\mkr(S)=\mkr(S')$`.
It is worth noting that the attacker has no direct control over the values `$x_5,x_6$` so this attack
 may not be useful in practice because for example the values `$x_5,x_6$` are not valid representation of balances.
Yet our security definition is broken, which is enough of a problem.

Next we will show another attack that is a bit more sophisticated as it allows the attacker to
 choose the values of the forged set `$S'$`.

<!--
More complex example where the attacker can choose his values, yet the hash functions is ad hoc.
-->

Let `$S = \{x_1,x_2,x_5,x_6\}$` where all the values are different and of length `$k+1$`.
Let $x_3'$ and $x_4'$ two values of length `$k$` chosen by the attacker.
Let `$S'= \{x_3,x_4\}$` where `$x_3=0||x_3'$` and `$x_4=0||x_4'$` two string of length `$k+1$`.
Let `$H: \{0,1\}^* \rightarrow \{0,1\}^k$` be a CRHF.
We build `$H':\{0,1\}^* \rightarrow \{0,1\}^{k+1}$` as follows:
<center>
	<table>
		<tr>
			<td>
				$H'(x)$
			</td>
			<td>
				$:=$
			</td>

			<td>
				$  \Huge\{ $
			</td>
			<td>
				\[
				\begin{array}{c}
					x_3=0||x_3' \text{ if }  x=x_1||x_2 \\
					x_4=0 ||x_4' \text{ if } x=x_5||x_6 \\
					1||H(x) \text{ otherwise} \\
				\end{array}
			\]
			</td>
		</tr>
	</table>
</center>

Let us prove first that if `$H$` is a CRHF then `$H'$` is also a CRHF.

Assume that an adversary is able to find a collision for `$H'$`, that is  `$x,x'$` different
such that `$r=H(x)=H(x')$`. Observe that `$H'$` is built in a way that the only input `$x$` with `$H'(x)=x_3$` is indeed
`$x=x1||x2$`. Similarly the only input `$x$` such that `$H'(x)=x_4$` is `$x=x_5||x_6$`.
This means that if $r$ starts with a `$0$`, then no collision is possible because
either `$x=x_1||x_2$` or `$x=x_5||x_6$` and by construction `$x_1||x_2$` is different from `$x_5||x_6$`.
This means that our adversary must try to find a collision with some value `$r$` starting with a `$1$`.
In this case (`$r$` starts with a `$1$`), the output of `$H'$` on `$x$` is of the form `$1||H(x)$`.
This means that the adversary has been able to find `$x,x'$` different such that `$1||H(x)=1||H(x')$`.
From this we can deduce that `$H(x)=H(x')$`.
So in summary we were able to prove that if there exist an adversary such that we can find
`$x,x'$` different while `$H'(x)=H'(x')$` then we can also find a collision for the function `$H$`.
However we assumed that `$H$` is a CRHF so this shall not happen. Hence if `$H$` is CRHF then `$H'$` is a CRHF as well.

So far so good, we just created another (strange) CRHF.

<figure>
  <img src="{{site.url}}/images/merkle-tree/tree-struct-attack.png" />
  <figcaption>Figure 4: A more sophisticated attack on the tree structure. The attacker can control values $x_3=0||x_3'$ and $x_4=0||x_4'$.</figcaption>
</figure>

However as in the previous attack, we can observe that the Merkle trees corresponding to `$S$` and `$S'$`
respectively have the same roots (see Figure 4).
Hence we have found a collision for our Merkle tree hash function as $\mkr(S)=\mkr(S')$.

One might object that this attack is artificial as well because indeed the adversary needs to build a custom CRHF
 based on values of the protocol and this might definitely not happen in practice.
However at least from a theoretical point of view we are not able anymore to prove that if `$H$` is a CRHF
 then the Merkle tree is also a CRHF (if we allow a flexible structure on the tree).
This is annoying indeed as in order to justify the security of a complex protocol one should first be able to validate the
 security of each component separately (note however this might not be sufficient but this is in all cases necessary).

On the practical side, this idea can be adapted to real world protocols with serious consequences.
Indeed in Bitcoin [an attack](https://bitslog.com/2018/06/09/leaf-node-weakness-in-bitcoin-merkle-tree-design/) related to
	the Merkle tree used to implement SPV (simple verification payments) was found by Sergio Lerner in 2017.
The attack is not trivial and involves quite some computational power, yet it could be performed in practice (the vulnerability has been fixed since).
The attack relies on two factors:
1. the flexible structure of the tree
2. the fact that values (Bitcoin transactions in this context) contain hashes of other values (i.e. Bitcoin transactions).
This helps the attacker to get more control in order to compute a hash (internal node) that is also a meaningful value for the protocol (Bitcoin transaction).

### Fix: padding techniques

So how can we avoid this kind of problem? The fix is indeed straightforward.
It consists of computing the root of the tree in a slightly different way.
Instead of returning the root `$r$` computed recursively by hashing the children, we return
`$H(r||W||H)$` where `$W$` is the width of the tree (number of leaves from left to right) and `$H$` is the height.
With this simple fix, the attacker cannot use the trick above anymore as the shape/structure of the tree is now *locked*
 by the last hash applied to obtain the root `$r'=H(r||W||H)$`.

## Adding new features to the Merkle tree: proof of liabilities

The functionality of Merkle trees can be extended in order to prove not only that some element belongs to a set but also other global predicates.
For example imagine we have a set of users $\{u_1,u_2,\cdots, u_n\}$ and each user $u_i$ has a balance $b_i$.
In addition to be able to prove user $u_i$ owns $b_i$ tokens, we may also want to convince a verifier
 that the total sum of balances is  $B = \sum_{i=1}^n b_i$.

This can be useful if we want to be sure that an exchange is transparent regarding the total
 money it owes to its customers. More on this below.

A way to achieve this requirement is to enrich each node of the tree with a balance value (see Figure 5).

<figure>
  <img src="{{site.url}}/images/merkle-tree/proof-of-liabilities.png" />
  <figcaption>Figure 5: Proof of liabilities. The Merkle tree tracks not only the elements of the set (balances assigned to users)
  but also the total sum of these balances.</figcaption>
</figure>

Computing the value of internal nodes consists of (1) computing the sum of the left and right child balances, (2) computing the hash of the concatenation of the new sum,
the left node hash value and the right node hash value.
By repeating this procedure until the root one obtains a hash value representing all the accounts and their balances, and a total sum owed to the customers.
The verification procedure will consist of:
 * recomputing the root, comparing it to the published one.
 * checking that each sum computed in the internal nodes is done correctly.
 * checking that all amounts in the tree are positive.

<figure>
  <img src="{{site.url}}/images/merkle-tree/proof-of-liabilities-privacy.png" />
  <figcaption>Figure 6: Proof of liabilities with enhanced privacy.
  The identity of the user in the leaves are not public.
  </figcaption>
</figure>

This idea first appeared shortly after MtGox's collapse in 2014,
when G. Maxwell proposed a protocol which aim is to enable an exchange to prove its solvency.
An important piece of the protocol is the *proof of liabilities* which we just described where the goal
 is to ensure that the exchange is accountable for all the customers' money it holds.
However, Maxwell's protocol is a bit different as each leaf contains the following information:
A balance `$B$`, and a hash value `$v=H(B||ID||Nonce)$` where `$ID$` is a unique identifier of the client and `$Nonce$` is a random value (see Figure 6).

Using the hash value instead of the balance data `$B||ID||Nonce$` directly into the leaf, is a way to increase users' privacy by not letting sibling leaf in the proof leak some information about another customer.

 <figure>
   <img src="{{site.url}}/images/merkle-tree/attack-proof-of-liabilities.png" />
   <figcaption>Figure 7: Attack on the proof of liabilities. Here we set $\Delta=10$. The forged proof provided to Alice contains
   the two nodes in red. A similar proof could be provided to other participants, adjusting the balance of the sibling leaf.</figcaption>
 </figure>

This extra feature however can make the whole construction vulnerable: as suggested in an [early post in Bitcoin Talk](https://bitcointalk.org/index.php?topic=595180.0) and described in more details in
this [technical report](https://eprint.iacr.org/2018/1139), it is possible for a malicious exchange to report less liabilities than required:
Instead of executing the protocol, the adversary will set the balances of the leaves' parents as follows:
if node `$N$`'s children are leaves `$L_1$` and `$L_2$` with respective balances `$B_1$` and `$B_2$`, then the balance of this internal
node is set to `$\mathtt{max}(B_1,B_2) + \Delta < \mathtt{sum}(B_1,B_2)$` instead of `$\mathtt{sum}(B_1,B_2)$`.
Then the malicious exchange will provide the authentication path to each user ensuring that the balance value of the sibling leaf is
 equal to the difference between the parent node and the user's leaf balance (see Figure 7).
On the example, we set $\Delta=10$.
The exchange will send a proof to Alice such that the sibling leaf has balance `$20$`, while Bob will
 receive a proof where the sibling leaf's balance is equal to `$10$`.
By doing this the exchange is able to fool its users as only `$100$` coins are declared as liabilities when in reality a total of `$140$`
 belong to the customers.
So what happened here? The problem is that the user has no way to verify that the balance claimed for the sibling leaf corresponds
to the balance used to compute the hash value. That is, given the sibling leaf `$B'$` , `$v=H(B||Id||Nonce)$`, nothing prevents the malicious
exchange to use different `$B$` and `$B'$` values. One way to fix the problem is to explicitly put the balance of each child instead
of only the sum in the parent node. Another more sophisticated approach is to provide a zero-knowledge proof that `$B'$` and `$B$` are equal
without revealing the preimage of `$v$`.

<br>

In our protocol will also rely on a Merkle trees for proving liabilities.
As you may expect we use padding techniques and ensure the attack described above does not work by revealing the balances of each customer in the proof as in Figure 5.
Indeed for this specific application maintaining the privacy of balances is not a concern as these balances are leaked anyways
 because users' deposits and withdrawals are made on-chain.
However we stumbled upon another trap.

## Another pitfall: sets are not dictionaries

In an early version of our protocol we assumed implicitly that the proof of liabilities scheme was a way to authenticate a dictionary and not a set.
Depending on the situation this can make a lot of difference.
Indeed the (fixed) proof of liabilities only ensures that each user can check that his
 balance has been considered in the global accounting and nothing else.
In particular nothing prevents a malicious exchange to create different leaves for the same client.
In the context of proving the solvency of an exchange this cannot be considered as an attack as the goal of the attacker is to minimize the total amount of liabilities.

However in our protocol these proofs are used to challenge potentially inconsistent statements of a corrupted exchange.
If we are not careful, the exchange could build two different proofs for the same customer in order to prove his integrity to the smart contract while not
 honouring his agreements with the customer.
So instead of changing the data structure (which is already quite tricky) we adapted the protocol so that a malicious operator
 could not use two different proofs for the same client.
This decision had two benefits: we avoided using a more complex data structure, and it turned out that the whole protocol became simpler after the modification.

## Conclusion and future work

As usual the evil is in the details and these attacks on apparently simple protocols are a reminder that intuition
 is not enough when dealing with security.
At Enuma Technologies we are always looking for better ways to build secure software.
Currently we are exploring formal methods techniques in order to mechanize the analysis and verification of our protocol.

## Author

[Philippe Camacho](https://scholar.google.com/citations?user=-7sqCHMAAAAJ&hl=en)

## Notes

<b id="f1">1</b> Note that the formal security definition is more complex. See chapter 8 of [Lecture Notes on Cryptography](http://cseweb.ucsd.edu/~mihir/papers/gb.pdf) by Shafi Goldwasser and Mihir Bellare. [↩](#a1)

## References

1. Antony, Mathis. Trustless, Noncustodial Exchange Prototype. [https://blog.enuma.io/update/2019/03/08/trustless-noncustodial-exchange.html](h
ttps://blog.enuma.io/update/2019/03/08/trustless-noncustodial-exchange.html), 2019

2. Sergio D. Lerner Leaf-Node weakness in Bitcoin Merkle Tree Design [https://bitslog.com/2018/06/09/leaf-node-weakness-in-bitcoin-merkle-tree-design/](https://bitslog.com/2018/06/09/leaf-node-weakness-in-bitcoin-merkle-tree-design/), 2017

3. Kexin Hu, Zhenfeng Zhang, Kaiven Guo. Breaking the Binding: Attacks on the MerkleApproach to Prove Liabilities and its Application
[https://eprint.iacr.org/2018/1139](https://eprint.iacr.org/2018/1139), eprint 2018

4. Zak Wilcox. Proving Your Bitcoin Reserves. [https://bitcointalk.org/index.php?topic=595180.0](https://bitcointalk.org/index.php?topic=595180.0), 2014.
