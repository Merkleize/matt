# MATT (Merkleize All The Things)

MATT is an approach for bitcoin smart contracts that is designed to be extremely simple, yet powerful.<br>
It could be enabled as a *soft fork* with various combinations of opcodes. The most direct way is based on an opcode called <tt>OP_CHECKCONTRACTVERIFY</tt> (<tt>CCV</tt>).

Key applications include:

- **Scaling and privacy** with more powerful layer 2 constructions.
- **Trustless bridges** for optimistic rollups or other sidechains.
- **Vaults** for better self custody.

By enabling scalable *fraud proof* protocols for arbitrary computations, it would bring permissionless innovation in the bitcoin's layer 2 space, while keeping Bitcoin's layer 1 simple and minimalistic.

**Start here**<br>
&nbsp;&nbsp;&nbsp;&nbsp;[Gentle introduction on StackExchange](https://bitcoin.stackexchange.com/questions/119239/what-are-matt-opcodes/119268#119268): *A beginner-friendly explanation of MATT and its core concepts.*<br>
&nbsp;&nbsp;&nbsp;&nbsp;[FAQ](FAQ.md) *Answers to common questions about MATT, its implementation, and use cases.*


**Specs and implementation**<br>
&nbsp;&nbsp;&nbsp;&nbsp;[BIP-443](https://github.com/bitcoin/bips/blob/master/bip-0443.mediawiki)<br>
&nbsp;&nbsp;&nbsp;&nbsp;[bitcoin-core implementation](https://github.com/bitcoin/bitcoin/pull/32080)<br>
&nbsp;&nbsp;&nbsp;&nbsp;[Discussion on the *amount* logic of <tt>OP_CHECKCONTRACTVERIFY</tt>](https://delvingbitcoin.org/t/op-checkcontractverify-and-its-amount-semantic/1527)<br>


**Presentations**<br>
[▶️ Merkleize All The Things](https://www.youtube.com/watch?v=56_rItUgrbA). *Advancing Bitcoin Conference − London, March 2nd, 2023*. [Slides](https://docs.google.com/presentation/d/1VCHJhXXzjn3qggQfNTZ3Ikiwi4QaXQYZzqkAea-QCBc/edit)<br>
*An introduction to MATT and how it enables scalable fraud proofs on Bitcoin.*

[▶️ sMATT Contracts Zero to Hero](https://youtu.be/BvXI1IOargk?si=dDi-7UdcO8OjCpGh). *btc++ − Austin, May 3rd, 2024*. [Slides](https://docs.google.com/presentation/d/1mcAJgPr7sBUZvT_0CBgY4NlkhUPiTpreUM4_YVx_g4o/edit)<br>
*A deep dive into MATT and the pymatt Python framework, with a re-implementation of [James O’Beirne's vaults](https://jameso.be/vaults.pdf) using <tt>CCV</tt>.*<br>
Timestamps:<br>
&nbsp;&nbsp;&nbsp;&nbsp;[Intro to MATT](https://youtube.com/watch?v=BvXI1IOargk&t=67s)<br>
&nbsp;&nbsp;&nbsp;&nbsp;[Vaults](https://youtube.com/watch?v=BvXI1IOargk&t=898s)<br>
&nbsp;&nbsp;&nbsp;&nbsp;[Vault demo](https://youtube.com/watch?v=BvXI1IOargk&t=1573s)<br>
&nbsp;&nbsp;&nbsp;&nbsp;[Code in pymatt](https://youtube.com/watch?v=BvXI1IOargk&t=1898s)
<br>

**Programming with MATT**
- [Docker image](https://github.com/Merkleize/docker) of bitcoin-inquisition + <tt>OP_CHECKCONTRACTVERIFY</tt> + <tt>OP_CAT</tt> + <tt>OP_CTV</tt> for regtest.
- [pymatt](https://github.com/Merkleize/pymatt) Python framework for MATT smart contracts, and examples.
<br>

**Research**
- [Games in the head (and fraud proofs for the plebs)](https://delvingbitcoin.org/t/games-in-the-head-and-fraud-proofs-for-the-plebs/446): an exploration on how to optimize 2-party fraud proof protocols implemented with MATT, and an estimate of the worst-case on-chain costs.
- [Aggregate delegated exit for L2 pools](https://delvingbitcoin.org/t/aggregate-delegated-exit-for-l2-pools/297): an approach for a fraud-proof based exit protocol for shared UTXOs that would allow many users to cooperate in order to withdraw their money.
- [elftrace](https://github.com/halseth/elftrace): Johan Torås Halseth's prototype of a compiler for arbitrary computation to RISC-V, and verify in Bitcoin Script via MATT's fraud proofs.
  - [▶️ Optimistic execution of RISC-V in Tapscript](https://youtu.be/byD4N2oLhFY?si=37E1oFbSVISngjo5): Presentation at *btc++ − Austin, May 1st, 2024*.

**bitcoin-dev**
- [Original bitcoin-dev post](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-November/021182.html)
  - [Example of fraud proof protocol instance](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-November/021205.html)
- [Vaults in the MATT framework](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-April/021588.html) and [code example](https://github.com/bitcoin-inquisition/bitcoin/compare/24.0...bigspider:bitcoin-inquisition:matt-vault)
- [MATT plays Rock-Paper-Scissors](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-May/021599.html) + [errata](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-May/021604.html)
- [Johan Torås Halseth](https://twitter.com/johanth)'s demonstration of [CoinPools exit clause](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-May/021719.html) using MATT and Merkle trees. [(docs)](https://github.com/halseth/tapsim/tree/matt-demo/examples/matt/coinpool) + [(code)](https://github.com/halseth/tapsim/blob/matt-demo/examples/matt/coinpool/script.txt)
