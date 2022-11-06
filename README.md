# General smart contracts in bitcoin via covenants
**Salvatore Ingala**

Covenants are UTXOs that are encumbered with restrictions on the *outputs* of the transaction spending the UTXO. More formally, we can define a *covenant* any UTXO such that at least one of its spending conditions is valid only if one or more of the outputs' `scriptPubKey` satisfies certain restrictions.

Generally, covenant proposals also add some form of introspection (that is, the ability for Script to access parts of the inputs/outputs, or the blockchain history).

In this note, we want to explore the possibilities unleashed by the addition of a covenant with the following properties:
- introspection limited to a single hash attached to the UTXO (the "covenant data"), and input/output amounts;
- pre-commitment to every possible future script (but not their data);
- few simple opcodes operating with the covenant data.

We argue that such a simple covenant construction is enough to extend the power of bitcoin's layer&nbsp;1 to become a **universal settlement layer** for arbitrary computation.

Moreover, the covenant can elegantly fit within P2TR transactions, without any substantial increase for the workload of bitcoin nodes.

*A preliminary version of these notes was presented and discussed at the [BTCAzores Unconference](https://btcazores.com/), on 23rd September 2022.*

# Preliminaries

We can think of a smart contract as a "program" that updates a certain *state* according to predetermined rules (which typically include access control by authorizing only certain public keys to perform certain actions), and that can possibly lock/unlock some coins of the underlying blockchain according to the same rules.

The exact definition will be highly dependent on the properties of the underlying blockchain.

In bitcoin, the only *state* upon which all the nodes reach consensus is the *UTXO set*; other blockchains might have other data structures as part of the consensus, like a key-value store that can be updated as a side effect of transaction execution.

In this section we explore the following concepts in order to set the framework for a definition of smart contracts that fits the structure of bitcoin:
- the contract's state: the "memory" the smart contract operates on;
- state transitions: the rules to update the contract's state;
- covenants: the technical means that can allow contracts to function in the context of a bitcoin UTXO.

In the following, an on-chain smart contract is always represented as a single UTXO that implicitly embeds the contract's state and possibly controls some coins that are "locked" in it. More generally, one could think of smart contracts that are represented in a *set* of multiple UTXOs; we leave the exploration of generalizations of the framework to future research.

## State
Any interesting "state" of a smart contract can ultimately be encoded as a list, where each element is either a bit, a fixed-size integers, or an arbitrary byte string.

Whichever the choice, it does not really affect what kinds of computations are expressible, as long as one is able to perform some basic computations on those elements.

In the following, we will assume without loss of generality that computations happen on a state which is a list of fixed length *S*&nbsp;=&nbsp;[*s*<sub>1</sub>,&nbsp;*s*<sub>2</sub>,&nbsp;...,&nbsp;*s*<sub>*n*</sub>], where each *s*<sub>i</sub> is a byte string.

### Merkleized state

By constructing a [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree) that has the (hashes of) the elements of *S* in the leaves, we can produce a short commitment *h*<sub>*S*</sub> to the entire list *S* with the following properties (that hold for a verifier that only knows *h*<sub>*S*</sub>):
- a (log&nbsp;*n*)-sized proof can prove the value of an element *s*<sub>*i*</sub>;
- a (log&nbsp;*n*&nbsp;+&nbsp;&#124;*x*&#124;)-sized proof can prove the new commitment *h*<sub>*S*'</sub>, where *S*' is a new list obtained by replacing the value of a certain leaf with *x*.

![A Merkle tree](/assets/merkletree.png)

This allows to compactly commit to a RAM, and to prove correctness of RAM updates.

In other words, a stateful smart contract can represent an arbitrary state in just a single hash, for example a 32-byte SHA256 output.

## State transitions and UTXOs

We can conveniently represent a smart contract as a *finite state machine* (FSM), where exactly one node can be active at a given time. Each node has an associated *state* as defined above, and a set of *transition rules* that define:
- who can use the rule;
- what is the next active node in the FSM;
- what is the state of the next active node.

It is then easy to understand how covenants can conveniently represent and enforce the smart contracts in this framework:

- The smart contract is instantiated by creating a UTXO encumbered with a covenant; the smart contract is in the initial node of the FSM.
- The UTXO's `scriptPubKey` specifies the current state and the valid transitions.
- The UTXO(s) produced after a valid transition might or might not be further encumbered, according to the rules.

Therefore, what is necessary in order to enable this framework in bitcoin Script is a covenant that allows the enforcement of such state transitions, by only allowing outputs that commit to a valid next node (and corresponding state) in the FSM.

It is not difficult to show that *arbitrary computation* is possible over the committed state, as long as relatively simple arithmetic or logical operations are available over the state.

*Remark*: using an acyclic FSM does not reduce the expressivity of the smart contracts, as any terminating computation on bounded-size inputs which requires cycles can be unrolled into an acyclic one.

### Merkleized state transitions

Similarly to how using Merkle trees allows to succinctly represent arbitrary data with a short, 32-byte long summary, the same trick allows to succinctly represent arbitrary state transitions (the smart contract's *code*) with a single 32-byte hash. Each of the possible state transitions is encoded as a Script which is put in a leaf of a Merkle tree; the Merkle root of this tree is a commitment to all the possible state transitions. This is exactly what the *taptree* achieves in Taproot (see [BIP-0341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki#constructing-and-spending-taproot-outputs)).

Later sections in this document will suggest a possible way of how both the contract's state and valid transition rules could be represented in UTXOs.

## On-chain computation?!

Should the chain actually *do* computation?

If naively designed, the execution of a contract might require a large number of transactions, which is not feasible.

While the covenant approach does indeed enable a chain of transactions to perform arbitrary computation, simple economic considerations will push protocol designers to perform any non-trivial computation off-chain, and instead use the blockchain consensus only to *verify* the computation; or, if possible, skip the verification altogether.

The fundamental fact that a blockchain's layer&nbsp;1 never *actually* needs to run complex programs in order to enable arbitrary complex smart contracting was observed in the past, for example in this [2016 post by Greg Maxwell](https://bitcointalk.org/index.php?topic=1427885.msg14601127#msg14601127).

Vitalik Buterin popularized the concept of *[functionality escape velocity](https://vitalik.ca/general/2019/12/26/mvb.html)* to signify the minimum amount of functionality required on layer&nbsp;1 in order to enable anything else to be built on top (that is, on layer&nbsp;2 and beyond).

In the following section, we will argue that a simple covenant construction suffices to achieve the functionality escape velocity in the UTXO model.

# Commitments to computation and fraud challenges

In this section, we explore how a smart contract that requires any non-trivial computation *f*&nbsp;:&nbsp;*X*&nbsp;↦&nbsp;*Y* (that is too expensive or not feasible with on-chain Script state transitions) can be implemented with the simple covenants described in the previous section.

The ideas in this section appeared in literature; the reader is referred to the references for a more comprehensive discussion.

We want to be able to build contracts that allow conditions of the type *f*(*x*)&nbsp;=&nbsp;*y*; yet, we do not want layer&nbsp;1 to be forced to perform any expensive computation.

In the following, we assume for simplicity that Alice and Bob are the only participants of the covenant, and they both locked some funds *bond*<sub>A</sub> and *bond*<sub>B</sub> (respectively) inside the covenant's UTXO.

1. Alice posts the statement "*f*(*x*)&nbsp;=&nbsp;*y*".
2. After a *challenge period*, if no challenge occurs, Alice is free to continue and unlock the funds; the statement is true.
3. At any time before the *challenge period* expires, Bob can start a challenge: "actually, *f*(*x*)&nbsp;=&nbsp;*z*".

In case of a challenge, Alice and Bob enter a challenge resolution protocol, arbitrated by layer&nbsp;1; the winner takes the other party's bond (details and the exact game theory vary based on the type of protocol the challenge is part of; choosing the right amount of bonds is crucial for protocol design).

The remainder of this section sketches an instantiation of the challenge protocol.

## The bisection protocol for arbitrary computation

In this section, we sketch the challenge protocol for an arbitrary computation *f*&nbsp;:&nbsp;*X*&nbsp;↦&nbsp;*Y*.

### Computation trace

Given the function *f*, it is possible to decompose the entire computation in simple elementary steps, each performing a simple, atomic operation. For example, if the domain of *x* and *y* is that of binary strings of a fixed length, it is possible to create a boolean circuit that takes *x* and produces *y*; in practice, some form of assembly-like language operating on a RAM might be more efficient and fitting for bitcoin Script.

In the following, we assume each elementary operation is operating on a RAM, encoded in the state via Merkle trees as sketched above.
Therefore, one can represent all the steps of the computation as triples *tr*<sub>*i*</sub>&nbsp;=&nbsp;(*st*<sub>*i*</sub>,&nbsp;*op*<sub>*i*</sub>,&nbsp;*st*<sub>*i*&nbsp;+&nbsp;1</sub>), where *st*<sub>*i*</sub> is the state (e.g. a canonical Merkle tree of the RAM) before the *i*-th operation, *st*<sub>*i*&nbsp;+&nbsp;1</sub> is the state after, and *op*<sub>*i*</sub> is the description of the operation (implementation-specific; it could be something like "add *a* to *b* and save the result in *c*).

Finally, a Merkle tree *M*<sub>*T*</sub> is constructed that has as leaves the values of the individual computation steps *T*&nbsp;=&nbsp;{*tr*<sub>0</sub>,&nbsp;*tr*<sub>1</sub>,&nbsp;...,&nbsp;*tr*<sub>*N*&nbsp;-&nbsp;1</sub>} if the computation requires *N* steps, producing the Merkle root *h*<sub>*T*</sub>. The height of the Merkle tree is ⌈log&nbsp;*N*⌉. Observe that each internal node commits to the portion of the computation trace corresponding to its own subtree.

Let's assume that the Merkle tree commitments for internal nodes are further augmented with the state *st*<sub>*start*</sub>, *st*<sub>*end*</sub>, respectively the state before the operation of in the leftmost leaf of the subtree, and after the rightmost leaf of the subtree.

### Bisection protocol

The challenge protocol begins with Alice posting what she claims is the computation trace *h*<sub>*A*</sub>, while Bob disagrees with the trace *h*<sub>*B*</sub>&nbsp;≠&nbsp;*h*<sub>*A*</sub>; therefore, the challenge starts at the root of *M*<sub>*T*</sub>, and proceeds in steps in order to find a leaf where Alice and Bob disagree (which is guaranteed to exist, hence the disagreement). Note that the arbitration mechanism knows *f*, *x* and *y*, but not the correct computation trace hash *h*<sub>*T*</sub>.

**(Bisection phase)**: While the challenge is at a non-leaf node of *M*<sub>*T*</sub>, Alice and Bob take turns to post the two hashes corresponding to the left and right child of their claimed computation trace hash; moreover, they post the start/end state for each child node. The protocol enforces that Alice's transaction is only valid if the posted hashes *h*<sup>*l*</sup><sub>*A*</sub> and *h*<sup>*r*</sup><sub>*A*</sub>, and the declared start/end state for each child are consistent with the commitment in the current node.

**(Arbitration phase):** If the protocol has reached the *i*-th leaf node, then each party reveals (*st*<sub>*i*</sub>,&nbsp;*op*<sub>*i*</sub>,&nbsp;*st*<sub>*i*&nbsp;+&nbsp;1</sub>); in fact, only the honest party will be able to reveal correct values, therefore the protocol can adjudicate the winner.

*Remark*: there is definitely a lot of room for optimizations; it is left for future work to find the optimal variation of the approach; moreover, different challenge mechanisms could be more appropriate for different functions *f*.

### Game theory (or *why the chain will not see any of this*)

With the right economic incentives, protocol designers can guarantee that playing a losing game always loses money compared to cooperating. Therefore, **the challenge game is never expected to be played on-chain**. The size of the bonds need to be appropriate to disincentivize griefing attacks.

### Implementing the bisection protocol's state transitions

It is not difficult to see that the entire challenge-response protocol above can be implemented using the simple state transitions described above.

Before a challenge begins, the state of the covenant contains the value of *x*, *y* and the computation trace computed by Alice. When starting the challenge, Bob also adds its claim for the correct computation trace, and the covenant enters the bisection phase.

During the bisaction phase, the covenant contains the claimed computation trace for that node of the computation protocol, according to each party. In turns, each party has to reveal the corresponding computation trace for both the children of the current node; the transaction is only valid if the hash of the current node can be computed correctly from the information provided by each party about the child nodes. The protocol repeats on one of the two child nodes on whose computation trace the two parties disagree (which is guaranteed to exist). If a leaf of *M*<sub>*T*</sub> is reached, the covenant enters the final arbitration phase.

During the arbitration phase (say at the *i*-th leaf node of *M*<sub>*T*</sub>), any party can win the challenge by providing correct values for *tr*<sub>*i*</sub>&nbsp;=&nbsp;(*st*<sub>*i*</sub>,&nbsp;*op*<sub>*i*</sub>,&nbsp;*st*<sub>*i*&nbsp;+&nbsp;1</sub>). Crucially, only one party is able to provide correct values, and Script can verify that indeed the state moves from *st*<sub>*i*</sub> to *st*<sub>*i*&nbsp;+&nbsp;1</sub> by executing *op*<sub>*i*</sub>. The challenge is over.

At any time, the covenant allows one player to automatically win the challenge after a certain timeout if the other party (who is expected to "make his move") does not spend the covenant. This guarantees that the protocol can always find a resolution.

### Security model

As for other protocols (like the lightning network), a majority of miners can allow a player to win a challenge by censoring the other player's transactions. Therefore, the bisection protocol operates under the *honest miner majority assumption*. This is acceptable for many protocols, but it should certainly be taken into account during protocol design.

# MATT covenants

We argued that the key to arbitrary, fully general smart contracts in the UTXO model is to use Merkle trees, at different levels:

1. succinctly represent arbitrary state with a single hash. *Merkleize the state!*
2. succinctly represent the possible state transitions with a single hash. *Merkleize the Script!*
3. succinctly represent arbitrary computations with a single hash. *Merkleize the execution!*

(1) and (2) alone allow contracts with arbitrary computations; (3) makes them scale.

*Merkleize All The Things!*

In this section we sketch a design of covenant opcodes that are taproot-friendly and could easily be added in a soft fork to the existing SegWitv1 Script.

## Embedding covenant data in P2TR outputs

We can take advantage of the double-commitment structure of taproot outputs (that is, committing to both a public key and a Merkle tree of scripts) to compactly encode both the covenant and the state transition rules inside taproot outputs.

The idea is to replace the internal pubkey *Q* with a key *Q'* obtained by tweaking *Q* with the covenant data (the same process that is used to commit to the root of the taptree). More precisely, if *d* is the data committed to the covenant, the covenant-data-augmented internal key *Q'* is defined as:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*Q'&nbsp;=&nbsp;Q&nbsp;+&nbsp;int(hash<sub>TapCovenantData</sub>(Q&nbsp;&#124;&#124;&nbsp;h<sub>data</sub>))G*

where *h<sub>data</sub>* is the sha256-hash of the covenant data. It is then easy to prove that the point is constructed in this way, by repeating the calculation.

If there is no useful key path spend, similarly to what is suggested in [BIP-341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki#constructing-and-spending-taproot-outputs) for the case of scripts with no key path spends, we can use the NUMS point *H = lift_x(0x0250929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0)*.

TODO: double check if the math above is sound

## Changes to Script

The following might be some minimal new opcodes to add for taproot transactions in order to enable the construction above. This is a very preliminary proposal, and not yet complete nor correct.
- `OP_SHA256CAT`: returns the SHA256 hash of the concatenation of the second and the first (top) element of the stack. (redundant if `OP_CAT` is enabled, even just on operands with total length up to 64 bytes)
- `OP_CHECKINPUTCOVENANTVERIFY`: let *x*, *d* be the two top elements of the stack; behave like `OP_SUCCESS` if any of *x* and *d* is not exactly 32 bytes; otherwise, check that the *x* is a valid x-only pubkey, and the internal pubkey *P* is indeed obtained by tweaking *lift_x(x)* with *d*.
- `OP_INSPECTNUMINPUTS`, `OP_INSPECTNUMOUTPUTS`, `OP_INSPECTINPUTVALUE` and `OP_INSPECTOUTPUTVALUE` - opcodes to push number on the stack of inputs/outputs and their amounts.
- `OP_CHECKOUTPUTCOVENANTVERIFY`: given a number *out_i* and three 32-byte hash elements *x*, *d* and *taptree* on top of the stack, verifies that the *out_i*-th output is a P2TR output with internal key computed as above, and tweaked with *taptree*. This is the actual covenant opcode.

TODO:
- Many contracts need parties to provide additional data; simply passing it via the witness faces the problem that it could be malleated. Therefore, a way of passing signed data is necessary. One way to address this problem could be to add a commitment to the data in the annex, and add an opcode to verify such commitment. Since the annex is covered by the signature, this removes any malleability. Another option is an `OP_CHECKSIGFROMSTACK` opcode, but that would cost an additional signature check.
- Bitcoin numbers in current Script are not large enough for amounts.

Other observations:
- `OP_CHECKINPUTCOVENANTVERIFY` and `OP_CHECKOUTPUTCOVENANTVERIFY` could have a mode where *x* is replaced with a NUMS pubkey, for example if the first operand is an empty array of bytes instead of a 32 byte pubkey; this saves about 31 bytes when no internal pubkey is needed (so about 62 bytes for a typical contract transition using both opcodes)
- Is it worth adding other introspection opcodes, for example `OP_INSPECTVERSION`, `OP_INSPECTLOCKTIME`? See [Liquid](https://github.com/ElementsProject/elements/blob/master/doc/tapscript_opcodes.md).
- Is there any malleability issue? Can covenants "run" without signatures, or is a signature always to be expected when using spending conditions with the covenant encumbrance? That might be useful in contracts where no signature is required to proceed with the protocol (for example, any party could feed valid data to the bisection protocol above).
- Adding some additional opcodes to manipulate stack elements might also bring performance improvements in applications (but not strictly necessary for feasibility).

*Remark:* the [additional introspection opcodes available in Blockstream Liquid](https://github.com/ElementsProject/elements/blob/master/doc/tapscript_opcodes.md) do indeed seem to allow MATT covenants; in fact, the opcodes `OP_CHECKINPUTCOVENANTVERIFY` and `OP_CHECKOUTPUTCOVENANTVERIFY` could be replaced by more general opcodes like the group \{`OP_TWEAKVERIFY`, `OP_INSPECTINPUTSCRIPTPUBKEY`, `OP_PUSHCURRENTINPUTINDEX`, `OP_INSPECTOUTPUTSCRIPTPUBKEY` \}.

### Variant: bounded recursivity

In the form described above, the covenant essentially allows fully recursive constructions (an arbitrary *depth* of the covenant execution tree is in practice equivalent to full recursion).

If recursivity is not desired, one could modify the covenants in a way that only allows a limited *depth*: a counter could be attached to the covenant, with the constraint that the counter must be decreased for `OP_CHECKOUTPUTCOVENANTVERIFY`. That would still allow arbitrary fraud proofs as long as the maximum depth is sufficient.

However, that would likely reduce its utility and prevent certain applications where recursivity seems to be a requirement.

The full exploration of the design space is left for future research.

# Applications

This section explores some of the potential use cases of the techniques presented above. The list is not exhaustive.

Given the generality of fraud proofs, some variant of every kind of smart contracts or layer two construction should be possible with MATT covenants, although the additional requirements (for example the capital lockup and the challenge period delays) needs to be accurately considered; further research is necessary to assess for what applications the tradeoffs are acceptable.

## State channels
A *state channel* is a generalization of a payment channel where, additionally to the balance at the end of each channel, some additional *state* is stored.
The state channel also specifies what are the rules on how to update the channel's state.

For example, two people might play a chess game, where the *state* encodes the current configuration of the board. The valid state transitions correspond to the valid moves; and, once the game is over, the winner takes a specified amount of the channel's money.

With eltoo-style updates, such a game could be played entirely off-chain, as long as both parties are cooperating (by signing the opponent's state update).

The role of the blockchain is to guarantee that the game can be moved forward and eventually terminated in case the other party does not cooperate.

In stateful blockchain, this is simply achieved by publishing the latest *state* (Merkleized or not) and then continuing the entire game on-chain. This is expensive, especially if the state transitions require some complex computation.

An alternative that avoids moving computations on-chain is the use of a challenge-response protocol, as sketched above.

Similarly to the security model of lightning channels, an honest party can _always_ win a challenge under the honest-majority of miners. Therefore, it is game-theoretically losing to attempt cheating in a channel.

## CoinPool

Multiparty state channels are possible as well; therefore, constructions like [CoinPool](https://coinpool.dev/v0.1.pdf) should be possible, enabling multiple parties to share a single UTXO.

## Zero knowledge proofs in L2 protocols

Protocols based on ZK-proofs require the blockchain to be the *verifier*; the verifier is a function that takes a zero-knowledge proof and returns true/false based on its correctness.

Instead of an `OP_STARK` operator in L1, one could think of compiling the `OP_STARK` as the function *f* in the protocol above.

Note that covenants with a bounded "recursion depth" are sufficient to express `OP_STARK`, which in turns imply the ability to express arbitrary functions within contracts using the challenge protocol.

One advantage of this approach is that no new cryptographic assumptions are added to bitcoin's layer&nbsp;1 even if OP_STARK does require it; moreover, if a different or better `OP_STARK2` is discovered, the innovation can reach layer&nbsp;2 contracts without any change needed in layer&nbsp;1.

## Optimistic rollups

John Light recently posted a [research report](https://bitcoinrollups.org) on how Validity Rollups could be added to bitcoin's layer&nbsp;1. While no exact proposal is pushed forward, the suggested changes required might include a combination of recursive covenants, and specific opcodes for validity proof verification.

Fraud proofs are the core for optimistic rollups; exploring the possibility of implementing optimistic rollups with MATT covenants seems a promising direction. Because of the simplicity of the required changes to Script, this might answer some of the [costs and risks](https://bitcoinrollups.org/#section-6-the-costs-and-risks-of-validity-rollups) analyzed in the report, while providing many of the same benefits. Notably, no novel cryptography needs to become part of bitcoin's layer&nbsp;1.

Optimistic Rollups would probably require a fully recursive version of the covenant (while fraud proofs alone are possible with a limited recursion depth).

# Acknowledgments

Antoine Poinsot suggested an improvement to the original proposed covenant opcodes, which were limited to taproot outputs without a valid key-path spend.

The author would like to thank catenocrypt, Antoine Riard, Ruben Somsen and the participants of the BTCAzores unconference for many useful discussions and comments on early versions of this proposal.

# References

The core idea of the bisection protocol appears to have been independently rediscovered multiple times. In blockchain research, it is at the core of fraud proof constructions with similar purposes, although not focusing on bitcoin or covenants; see for example:

- [*Harry Kalodner et al.* "Arbitrum: Scalable, private smart contracts." − *27th USENIX Security Symposium*. 2018.](https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-kalodner.pdf)
- [*Jason Teutsch and Christian Reitwießner.* "A scalable verification solution for blockchains" − TrueBit protocol. 2017](https://people.cs.uchicago.edu/~teutsch/papers/truebit.pdf)

The same basic idea was already published prior to blockchain use cases; see for example:

- [*Ran Canetti, Ben Riva, and Guy N. Rothblum.* "Practical delegation of computation using multiple servers." − *Proceedings of the 18th ACM conference on Computer and communications security*. 2011.](http://diyhpl.us/~bryan/papers2/bitcoin/Practical%20delegation%20of%20computation%20using%20multiple%20servers.pdf)
