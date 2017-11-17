
Drivechain Documentation
===========================

    Paul Sztorc 
	August 15, 2017
	Document 1 of 3
	v0.2

Note from Paul: Double brackets "{{" and "}}" surround parts that are "unfinished".
???'s indicate free parameters that could be anything, and have yet to be determined.


Two Concepts
==============

For clarity, I've broken the drivechain idea into two modular concepts: 

1. The "hashrate escrow" which allows users to make their transactions vulnerable to reduced-security SPV proofs.
2. The "blind merged mining" idea which is basically a modern version of merged-mining that is specialized in order to make asymmetric sidechains as tame as possible.

Also for clarity, this documentation intentionally does *not* cover anything done on *any* sidechain. It focuses completely on the mainchain.

Each of the two concepts [a] tracks data that is elsewhere in the block, and rearranges it, and [b] has its own transaction type. The new transaction-types are comparable to similar soft forks: to old nodes they are always valid spends, and upon this canvas we instruct new nodes to invalid these tx-types unless a very strict set of rules are followed.

A total of three hash-commitments are required, arranged in the following way:

               ** Storage of Hash Commitments **   

       Hashrate Escrow Items  ----> (mandatory) inserted in Coinbase at TxOut index 0
          /           \            
         /             \                       
    Sidechain_DB   Withdrawal_DB      

	Blind Merged-Mining Data  ----> (optional) inserted in Coinbase at some TxOut, (identified by a 4-byte header)
	

This first commitment must always be present. This is to prevent free-riding. However, it is not very burdensome -- if miners don't want to participate, they can easily calculate a default valid hash-values. These do not depend on any information they would not already have.

If the second commitment is not present, sidechain nodes will simply assume that no side:block was found in the last 10 minutes. There is no possibility of free-riding here, as the commitment does not affect anything that is happening across multiple blocks.

(An earlier version of this document described a way of combining the two ideas into a single hash. I decided that it was confusing people, and have removed it.)


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

