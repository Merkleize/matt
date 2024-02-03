# Frequently Asked Questions (FAQ)

1. [General Questions](#general-questions)
   - [What is MATT?](#what-is-matt)
   - [What is `OP_CHECKCONTRACTVERIFY`?](#what-is-op_checkcontractverify)
   - [Are there other ways to enable MATT constructions?](#are-there-other-ways-to-enable-matt-constructions)
   - [Does MATT make bitcoin's Script Turing-complete?](#does-matt-make-bitcoins-script-turing-complete)
   - [Exciting! How can I write MATT smart contracts?](#exciting-how-can-i-write-matt-smart-contracts)
2. [Fraud Proofs](#fraud-proofs)
   - [What is a fraud proof?](#what-is-a-fraud-proof)
   - [Do MATT contracts need fraud proofs?](#do-matt-contracts-need-fraud-proofs)
   - [Do MATT fraud proofs actually scale?](#do-matt-fraud-proofs-actually-scale)
   - [Could future upgrades of bitcoin Script make fraud proofs unnecessary?](#could-future-upgrades-of-bitcoin-script-make-fraud-proofs-unnecessary)
   - [What are the trade-offs of fraud proofs?](#what-are-the-trade-offs-of-fraud-proofs)
   - [Can miners just steal from fraud proof protocols?](#can-miners-just-steal-from-fraud-proof-protocols)
3. [Design Choices](#design-choices)
   - [Does MATT allow recursive covenants?](#does-matt-allow-recursive-covenants)
   - [Could the "data" of MATT contracts be committed somewhere else?](#could-the-data-of-matt-contracts-be-committed-somewhere-else)
4. [Related Work](#related-work)
   - [How does MATT compare with BitVM?](#how-does-matt-compare-with-bitvm)


## General Questions


### What is MATT?

MATT is a minimalistic approach to smart contracts designed to require only simple, non-invasive changes to Bitcoin consensus rules, yet bring a large increase in expressive power.

This is achieved by means of an extensive use of [Merkle trees](https://en.wikipedia.org/wiki/Merkle_tree), with multiple purposes. MATT is an acronym of *Merkleize All The Things*.

The core new Script primitive introduced in MATT is the ability to:
- force an output to have a certain Script, and control its amount
- attach a small piece of data to an output
- 'read' the data of an input (usually the current one)

The second ingredient which is necessary in order to achieve generality is the ability to "compress" multiple pieces of data into single 32-byte hash, which can be achieved with Merkle trees.

There are many ways to introduce the MATT functionality with new opcodes, and constructions based on MATT are easy to port to any approach that provides the necessary functionality.


### What is `OP_CHECKCONTRACTVERIFY`?

`OP_CHECKCONTRACTVERIFY` is a proof-of-concept of an opcode that directly implement the core functionality of MATT. It is designed to be clean and generic, and to work well with P2TR (taproot) scripts.

In short, `OP_CHECKCONTRACTVERIFY` can be summarized as follows:  *Check that an input/output is a P2TR output whose public key is obtained by tweaking a certain key with `<data>`, and then with `<taptree>`*. The opcode can also constrain the output amount - in the most common case, by forcing all the amount of the current input to be preserved in an output.

A formalization of the semantic of the opcode is [here](https://github.com/ariard/bitcoin-contracting-primitives-wg/issues/25#issuecomment-1595762674).

Together with `OP_CAT`, it's a complete, and straightforward implementation of the MATT proposal.

Additional opcodes could be useful to make constructions better or more efficient, but they are not strictly required for MATT.


**Warning:** Note that the current `OP_CHECKCONTRACTVERIFY` implementation is a proof-of-concept and has not received extensive testing and review. It has the goal to develop the framework and experiment with arbitrary MATT contracts, but it does not satisfy the quality standards for a soft-fork proposal.


### Are there other ways to enable MATT constructions?

Yes, the core functionality of MATT is quite basic, and it's possible to obtain it by combining several general-purpose opcode. For example, the following opcodes are enough:

- `OP_CAT` (concatenate two elements)
- introspection of input/output script and amounts
- `OP_TWEAK` (scalar multiplication of a secp256k1 point)

Of course, optimizing the opcodes to the typical pattern of MATT contracts helps its usability and the script size of the constructions using it.


### Does MATT make bitcoin's Script Turing-complete?

No, Script remains simple and *not* Turing-complete.

MATT only enables UTXOs to enforce simple state machines, where each state transition (corresponding to a bitcoin transaction) is simple enough to be computed (or, rather, verified), in Script.

Very long state machines could in theory be used to compute any function; however that wouldn't be feasible in practice for complicated functions.

However, a rather simple state machine is enough to implement a *fraud proof protocol* for arbitrary computation.

Therefore, while Script will still only be able to verify relatively simple predicates, it enables smart contracts that depend on the result of arbitrary computation.

Thanks fraud proofs, one can argue that *any possible technology that is interesting for bitcoin could be implemented as a layer two*. That holds even for applications that require cryptographic techniques that are not available to bitcoin's layer 1.

Of course, fraud proofs come with trade-offs that need to be assessed in the context of each specific application.


### Exciting! How can I write MATT smart contracts?

[pymatt](https://github.com/Merkleize/pymatt) is an experimental framework to write smart contracts using the MATT approach. Together with the [docker container](https://github.com/Merkleize/docker), you can get started writing and testing smart contracts on `regtest` in no time.


## Fraud Proofs

### What is a fraud proof?

The idea of fraud proofs is to replace a (possibly expensive) computation in a smart contract with just a claim that a computation would give a certain result.

As long as the interested parties in the contract have a way of disproving fraudulent claims, and the contract is constructed in such a way that fraud attempts are punished, then frauds (and fraud proofs) are never expected to actually end up on chain.

The saving generally comes not only from avoiding an expensive operation in Script, but also by not putting the entire witness data that the computation would need: a commitment to it is generally sufficient, as long as the counterparties are able to independently compute the correct data themselves.

The trade-off of using protocols based on fraud proofs (optimistic protocols) in comparison to a full on-chain validation of the contract transition are:

- fraud proofs require a challenge period, in comparison to direct computations that are confirmed in a single transaction; moreover, some capital lockup is generally needed during the challenge period;
- fraud proof protocols rely on the assumption that miners are not actively censoring the transactions of a party in the smart contract (and blocks containing them).

The most commonly used optimistic protocol in bitcoin today is in a Lightning channel: the “justice transaction” that is published if a counterparty puts an old state on-chain is indeed a fraud proof.


### Do MATT contracts need fraud proofs?

No!

Fraud proof protocols are one of the things that can be built with MATT¸ and can be used as a building block for other smart contracts. They greatly expand the realm of what's possible in Bitcoin Script. 

In a certain sense, *any possible layer-2 system can be implemented on Bitcoin using MATT*, thanks to fraud proofs.

However, constructions that do not require fraud proofs if implemented with other covenant proposals are likely to not require fraud proofs if implemented with MATT.


### Do MATT fraud proofs actually scale?

Yes!

A well designed smart contract based on fraud proofs can make sure that actually executing a fraud proof on-chain is never expected to happen. That's because uncooperative parties are financially punished, making honest behaviour always the cheapest option.

Of course, it is still important that fraud proofs *can* be brought and completed on chain if needed.

A ballpark estimate based on a current, unoptimized implementation suggests that a MATT fraud proof will never require more than 15000 vbytes (for a computation with 2^32 = 4 billion steps!), and it's reasonable to expect that, in practice, the total size would rarely go beyond a few thousands.


### Could future upgrades of bitcoin Script make fraud proofs unnecessary?

Smart contract protocols that rely on fraud proofs rather than full on-chain verification are called *optimistic*.

Optimistic smart contract have the inherent advantage of allowing on-chain settlement to depend on state changes that happen entirely off-chain. For example, in a lightning channel, one has a reasonable guarantee that a payment has gone through without any on-chain transaction: they obtained the necessary signatures/preimages from the counter-party and they know they can win an on-chain challenge if necessary.

This is simply not possible without some kind of challenge mechanism, as the statements that need to be validated depend on data that was never stored (nor committed to) on-chain.


### What are the trade-offs of fraud proofs?

The trade-offs depend on the specific application fraud proofs are used for.

Generally speaking, some of the common trade-offs of optimistic contracts are:

- *Capital lockup*: funds are only available after a challenge period, rather than immediately. Moreover, optimistic protocols usually require some additional capital to be locked in a contract, in order to keep participants honest and make *griefing* expensive for dishonest participants.
- *Censorship resistance assumption*: Optimistic protocols rely on censorship resistance at layer 1, since the inability to get transactions confirmed in time would lead to loss of funds.


### Can miners just steal from fraud proof protocols?

A single miner or entity controlling less than 50% of the hashrate only has a slight advantage if they are a party in a challenge protocol: even if they censor the counterparty's transactions, some other miner will have an incentive to confirm the other party's transactions if they pay high-enough fees.

A full blown 51% attack, where the attackers censor the other side's transactions, *and reject any block that includes them*, can win all challenges. That differs from regular on-chain transactions, where even a hashrate majority can permanently censor, but not steal funds.

This is an inherent tradeoff of optimistic protocols.

The degree of how worrisome this attack vector is depends on at least two factors:
- *Challenge openness*: if only a few parties in the contract can initiate a challenge, the 51% attack is only an issue if the counterparties collude with the offending miners. Protocols where anyone can start a fraud proof might be easier to attack.
- Stakes: contracts with much higher stakes would increase the reward for a successful attack.

The lightning network is vulnerable to this attack, but sits at the lowest end of the spectrum: only the counterparty can attempt to fraudulently win a challenge, and the amount locked in a single lightning channel is usually not extremely high. Therefore, a wannabe-51%-attacker would probably need to prepare an attack over a long time, locking capital in many different lightning channels, and attack all of them simultaneously.

On the opposite end of the spectrum, a hypothetical *optimistic sidechain* locking a large amount of capital, with completely open access and a 2-way peg based on fraud proofs would be most vulnerable to a 51% attack.


## Design choices


### Does MATT allow recursive covenants?

This depends on how it is implemented. The current implementation of `OP_CHECKCONTRACTVERIFY` does allow recursion. While many smart contracts do not need recursion, it is useful for some of them. For example, the [vaults](https://github.com/Merkleize/pymatt/tree/master/examples/vault) demo uses it to allow partial revaulting - a nice-to-have, rather than an essential feature.

However, it is possible to design opcodes that enable the functionality of MATT without unbounded recursion. A limited recursion depth of a few tens (or perhaps less) transactions is sufficient to implement fraud proof protocols for arbitrary computation.


### Could the "data" of MATT contracts be committed somewhere else?

Yes, there are many possible approaches, and they make different tradeoffs.

`OP_CHECKCONTRACTVERIFY` uses a double *tweak* in order to commit to the data.

A tweak is a technique

Pros:
- The approach remains fully compatible with Pay-2-Taproot UTXOs.
- The size of the UTXOs is not increased.
- The cost of revealing the data is only paid for Scripts that require this data.
- It is rather elegant to not mix the contract *data* with the *program* (the taptree).

Cons:
- A tweak is computationally a rather expensive operation.

An alternative approach could be to embed the data as a "fake" tapleaf in the taptree. However, if a leaf next to the root is used, the cost of using *every* tapscript increases (regardless if they access the data or not); if a deep tapleaf is used, then the cost of revealing the data increases substantially for contracts with multiple tapleaves. However, this approach would avoid the expensive data tweak. 

Another approach could be a completely new SegWit version, which would allow more freedom in how to design the features required for MATT; however, this would dramatically increase the size of the required code changes, and it is unclear if it would have substantial benefits.


## Related work

### How does MATT compare with BitVM?

[BitVM](https://bitvm.org/bitvm.pdf) is the remarkable discovery that a channel (among two or more fixed parties) that allows fraud proofs for arbitrary computations is possible today in bitcoin. Such channels are known in academic literature as *state channels*, and they can be considered a generalization of lightning channel where the two parties are able to play an arbitrary game. The game can be played entirely off-chain if there's cooperation; otherwise, the other parties can *force* the game to be played on-chain.

However, MATT is more general, as it enables contracts that implement arbitrary on-chain state machines, rather than just *channels*.

In fact, something as simple as an on-chain [vault](https://github.com/Merkleize/pymatt/tree/master/examples/vault) construction is not possible with BitVM, while it is enabled by an explicit soft-fork for covenants.

The lack of covenants comes at a hefty cost: BitVMs channel require a large precomputation, with significant storage and bandwidth requirements prior to setup.

State channels implemented with MATT require no precomputation whatsoever, and an on-chain execution of a fraud proof protocol is expected to be orders of magnitude cheaper than with BitVM.

Read a more detailed comparison in [this Twitter thread](https://twitter.com/salvatoshi/status/1712772143753408824).
