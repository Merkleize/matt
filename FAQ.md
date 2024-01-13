# Frequently Asked Questions (FAQ)

## General Questions

### What is MATT?

MATT is a minimalistic approach to smart contracts designed to require only simple, non-invasive changes to Bitcoin consensus rules, yet bring a large increase in expressive power.

It achieves so with an extensive use of [Merkle trees](https://en.wikipedia.org/wiki/Merkle_tree), with multiple purposes. MATT is an acronym of *Merkleize All The Things*.

The core new Script primitive introduced in MATT is the ability to:
- force an output to have a certain Script, and control its amount
- attach a small piece of data to an output
- 'read' the data of the an input (usually the current one)

The second ingredient which is necessary in order to achieve generality is the ability to "compress" multiple pieces of data into single 32-byte hash, which can be achieved with Merkle trees.

There are many ways to introduce the MATT functionality with new opcodes, and constructions based on MATT are easy to port to any approach that provides the necessary functionality.

### What is `OP_CHECKCONTRACTVERIFY`?

`OP_CHECKCONTRACTVERIFY` is a proof-of-concept of an opcode that directly implement the core functionality of MATT. It is designed to be clean and generic, and to work well with P2TR (taproot) scripts.

In short, `OP_CHECKCONTRACTVERIFY` can be summarized as follows:  *Check that an input/output is a P2TR output whose public key is obtained by tweaking a certain key with `<data>`, and then with `<taptree>`*. The opcode can also constrain the output amount - in the most common case, by forcing all the amount of the current input to be preserved in an output.

A formalization of the semantic of the opcode is [here](https://github.com/ariard/bitcoin-contracting-primitives-wg/issues/25#issuecomment-1595762674).

Together with `OP_CAT`, it's a complete, and straightforward implementation of the MATT proposal.

Additional opcodes could be useful to make constructions better or more efficient, but they are not strictly required for MATT.


**Warning:** Note that the current `OP_CHECKCONTRACTVERIFY` implementation is a proof of concept and has not received extensive testing and review. It has the goal to develop the framework and experiment with arbitrary MATT contracts, but it does not satisfy the quality standards for a soft-fork proposal.

### Are there other ways to enable MATT constructions?

Yes, the core functionality of MATT is quite basic, and it's possible to obtain it by combining several general-purpose opcode. For example, the following opcodes are enough:

- `OP_CAT` (concatenate two elements)
- introspection of input/output script and amounts
- `OP_TWEAK` (scalar multiplication of a secp256k1)

Of course, optimizing the opcodes to the typical pattern of MATT contracts helps its usability and the script size of the constructions using it.

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

No! Constructions that have been proposed by building on other covenant proposals do not need fraud proofs if implemented with MATT.

Fraud proof protocols are one of the things that can be built with MATT¸ and can be used as a building block for other smart contracts. They greatly expand the realm of what's possible in Bitcoin Script. 

In a certain sense, *any possible layer-2 system can be implemented on Bitcoin using MATT*, thanks to fraud proofs.

However, constructions that do not require fraud proofs if implemented with other covenant proposals are likely to not require fraud proofs if implemented with MATT.

### Do MATT fraud proofs actually scale?

Yes!

A well designed smart contract based on fraud proofs can make sure that actually executing a fraud proof on-chain is never expected to happen. That's because uncooperative parties are financially punished, making honest behaviour always the cheapest option.

Of course, it is still important that fraud proofs *can* be brought and completed on chain if needed.

A ballpark estimate based on a current, unoptimized implementation suggests that a MATT fraud proof will never require more than 15000 vbytes (for a computation with 2^32 = 4 billion steps!), and it's reasonable to expect that, in practice, the total size would rarely go beyond a few thousands.

