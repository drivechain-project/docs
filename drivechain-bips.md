
Drivechain Documentation
===========================

    Paul Sztorc 
	August 15, 2017
	Document 1 of 3

Note from Paul: Double brackets "{{" and "}}" surround parts that are "unfinished".


Two Concepts
==============

For clarity, I've broken the drivechain idea into two modular concepts: 

1. The "hashrate escrow" which allows users to make their transactions vulnerable to reduced-security SPV proofs.
2. The "blind merged mining" idea which is basically a modern version of merged-mining that is specialized in order to make asymmetric sidechains as tame as possible.

Also for clarity, this documentation intentionally does *not* cover anything done on *any* sidechain. It focuses completely on the mainchain.

Each of the two concepts [a] tracks data that is elsewhere in the block, and rearranges it, and [b] has its own transaction type. The new transaction-types are comparable to similar soft forks: to old nodes they are always valid spends, and upon this canvas we instruct new nodes to invalid these tx-types unless a very strict set of rules are followed.

A total of three hash-commitments are required, arranged in the following way:

               ** Storage of Hash Commitments **   

                 Root of New Merkle Tree  ----> ( inserted into Index 1 of a Coinbase txn {{if the value of TxOut is zero?}} )
                    /               \
                   /                 \
                  /                   \
                 /                     \
       Hashrate Escrow Items            \
          /           \            Blind Merged-Mining Data 
         /             \             
    Sidechain_DB   Withdrawal_DB      




This commitment must always be present, but if miners wish to decline to participate in these new subsystems, they can calculate new, valid hash-values that do not depend on any information they do not already have.

To maintain the modularity of the two proposals, each individual proposal can require two hash commitments, in which the second (non-specified) hash is free to be anything.

In other words, the "Hashrate Escrow" proposal [left side] would require a hash commitment of hash(itself, X) where X can take on any value. And the "Blind Merged-Mining" proposal [right side] would require a hash commitment of hash(Y, itself), where Y could take on any value. Thus they are fully separable.

{{ Or, we can store "Hashrate Escrow" and "BMM" hash commitments each in their own Coinbase TxOut. It doesn't matter very much. }}


Links
======

1. [Link to 1st BIP](https://github.com/drivechain-project/docs/blob/master/bip1-hashrate-escrow.md) -- Hashrate Escrow
2. [Link to 2nd BIP](https://github.com/drivechain-project/docs/blob/master/bip2-blind-merged-mining.md) -- Blind Merged-Mining




 === END OF DOCUMENTATION ===


### TODO (meta)


Plans for this document:

1. Edit for clarity.
2. Flesh out the "soft" sections ie "Motivation" etc
3. Somehow reconcile this with more detailed descriptions (ie txdb.cpp [1] ?)

[1]  https://github.com/bitcoin/bitcoin/blob/master/src/txdb.cpp

