
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

       Hashrate Escrow Items  ----> to Coinbase (index 1?)
          /           \            
         /             \                       
    Sidechain_DB   Withdrawal_DB      

	Blind Merged-Mining Data  ----> to Coinbase (index 2?)
	
	
{{ Perhaps these could be inserted into indexes 1 and 2; more info: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-June/014669.html }}

This first commitment must always be present, but if miners wish to decline to participate, they can easily calculate a valid hash-values (that does not depend on any information they would not already have).

If the second commitment is not present, sidechain nodes will simply assume that no side:block was found in the last 10 minutes.

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

