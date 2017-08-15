Drivechain Documentation
Paul Sztorc 
August 15, 2017
Document 3 of 3
===========================

Note from Paul: Double brackets "{{" and "}}" surround parts that are "unfinished".



Concept Two: Blind Merged-Mining
==================================


Again, this focuses only on the tasks that mainchain Bitcoin must perform.

For a description of functions each sidechain must perform, [this](http://www.truthcoin.info/images/bmm-outline.txt) may be more helpful.

For a description of the "whole picture", [this](http://www.truthcoin.info/blog/blind-merged-mining/) may be most helpful.


### Motivation

Merged-Mining (MM) allows miners to reuse their hashing work to secure other chains (for example, as in Namecoin). However, MM has two drawbacks:

1. Miners must run a full node of the other chain (because they need to assemble a valid block to work on).
2. Miners are paid on the other chain, not on the regular BTC mainchain.

Blind Merged-Mining (BMM) attempts to address those shortcomings.


### Description

The goals are to [1] track a set of ordered hashes (the merged-mining), and allow miners to "sell" the act of finding a sidechain block (through the use of a new OP CODE -- )

Specifically, we track 35 bytes of information, per sidechain:

    1-byte   - Sidechain Number (known as "Account Number" in hashrate-escrows.md)
    32-bytes - prevSideBlockHash (also known as "h*")
    2-bytes  - blockMod (a counter -- in pratice, it is the sidechain's block number modulus 65535)


For 256 sidechains this amounts to 8.96 KB per block.



### Creating / Broadcasting This Data


#### Creation

The sidechain number is globally known.

The other two items are created by sidechain nodes -- they build valid sidechain block-headers, of which they compute the hash and block number (and therefore, the blockMod).


#### Broadcast

Where should this data be located, in blocks?

{{ There are at least two ways.}}


Method 1: Put each in a coinbase OP RETURN

    1-byte - OP_RETURN (0x6a)
    1-byte - Push the following 39 bytes (0x27)
    4-byte - Message header (0x????????)
    35-byte - The three items directly mentioned above



Method 2: Use an Ordered Merkle Tree of Known Length, and put the root in a coinbase OP RETURN


{{ This assumes that BMM will inherent D1 (from hashrate-escrow.md). }}

1. For each account that is added to D1, nodes construct an ordered tree and commit to its root.
2. Each leaf of the tree consists of 34 bytes, which are all zero by default. These zeros indicate that no block was merged-mined.


                     Blind Merged-Mining Data
                           /          \
                          /            \
                         /\            /\
                        /  \          /  \
                       /    \        /    \
                      h1    h2     ...    h4
                      /     /       /     /
                           /
                          /
             hash(h*, BlockMod) of Account 2


By assuming that the tree is ordered, we save 1 byte per sidechain per block. We also know exactly how long the tree is.



### Using This Data

#### The Need for a "Rachet"

See here: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-July/014789.html


#### D3 - Blind Merged Mining New Database

Ultimately, the mainchain full nodes need to have, per sidechain, an ordered list of the most recent 4000 (h*, blockMod) pairs.

For the most recent X=4000 blocks, upgraded nodes will track:

1. blockNum -- This (mainchain) block number.
2. qSidechains -- Total quantity of sidechains in existence, at this time.
3. "Linking Data" (How the h*s are located):
    3a. merklePath -- The path from the mainchain blockheader's hashMerkleRoot down to its coinbase txn.
    3b. coinbase -- The contents of the coinbase txn itself.
    3c. merkleTree (if Method 2, above) -- all of the hashes in the "BMMD Tree" (above)
4. Per sidechain:
	4a. Location of (h*, blockMod) pair -- index of coinbase txn (Method 1) or merkleRoot (Method 2)
    4a. Data itself:
        4ai.  h* (prevSideBlockHash)
        4aii. blockMod


This results in D3, which might look something like: 
       
         SC 1       SC 2       SC 3      
         ----      ------     ------
   1.   h*, 746   h*,   25    h*, 1156
   2.   h*, 747   h*,   26    h*, 1157
   3.   h*, 748   h*,   27    h*,  903
   4.   h*, 749   h*,   28    h*,  904
  ...
3999.   h*, 744   h*, 4023    h*, 3487 
4000.   h*, 745   h*, 4024    h*, 3488



#### New Validation Rules

Each mainchain node tracks, per sidechain, the last 4000 blockMods. Each *new* block is not permitted to add an entry to this list, unless it is within range (-4000, +1) of the most recent blockMod.


  {{ Note: The trick is that sidechains [1] know exactly where to look for these hashes (and, therefore, will always be able to find one unique has per mainchain block) and [2] accept this hash as being equal to "a prevBlockHash which meets the chain's difficulty target". }}


### OP CheckBribeVerify

This section reappropriates an OP NOP, so as to allow miners to "sell" the act of mining a sidechain block. This allows all miners to benefit from sidechains equally -- even those which are unable to run burdensome sidechain nodes.

{{ I actually can't remember most of what we decided, for this. }}

First, we define "Mary" as a mainchain miner, and "Simon" as a sidechain node.

The goal is to construct a payment from Simon to Mary:

1. If conditions_1 are met, Mary can claim the money with finality.
2. If, instead, the timeout_condition is met, Simon can reclaim the money.

{{ See https://lightning.network/  -- for info about how this is often done. }}


The TxOut in question would include three arguments: (h*, this_blocknumber, timeout).

**Mary** would be able to spend it if she can prove two things:

1. ..that a sidechain block of hash h* was mined recently, AND
2. ..that the sidechain block has been buried by 200 side-blocks.

The first is proved by using the linking data to show that a (h*,bM) pair exists in D3. The second is shown by showing that a (h*,bM) pair exists in {{the previous}} block (ie, one block ago), and that the bM value of this pair is at least 200 greater than the bM value of the pair referenced in the previous sentence.

In contrast, **Simon** is able to spend the txn (and recapture his money) if Mary goes some length of time (say, 500 blocks) without accomplishing what she said she would be able to accomplish.

As usual, Simon and Mary can utilize the lightning network, such that their txns never needs to make it onto the chain, unless there is a dispute.

To discourage Mary from locking up Simon's money in an inconvenient way, we could force Mary to also send a very small amount of her money into the transaction -- much less than she is being paid, but enough to reimburse Simon for the inconvenience of having his money locked away temporarily.







{{ BIP Stuff }}


Header
=======

  BIP: ????
  Layer: Consensus (soft fork)
  Title: Blind Merged-Mining (Consensus layer)
  Author: Paul Sztorc <truthcoin@gmail.com>
          CryptAxe <cryptaxe@gmail.com>
          Chris Stewart <chris@suredbits.com>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-??????
  Status: Draft
  Type: Standards Track
  Created: 2017-08-14
  License: PD



Abstract
==========

{{ Abstract }}


Motivation
============

{{ Take from the intro -- "Merged...those shortcomings" }}


Backward compatibility
========================

{{ Usual stuff about soft forks. }}

{{ What old nodes do if they encounter new data. }}

BONUS: Sidechains inherit the "soft fork" safe-upgrade-incentive, as soft forks are the only way to guarantee that the WT^s reported by different clients will continue to match.


Deployment
===========

{{ Stuff about versionbits -- copy from https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#deployment }}


Reference Implementation
==========================

See: https://github.com/drivechain-project/bitcoin/tree/mainchainBMM

Also, for interest, see an example sidechain here: https://github.com/drivechain-project/bitcoin/tree/sidechainBMM


References
============

* http://www.drivechain.info/literature/index.html
* http://www.truthcoin.info/blog/blind-merged-mining/
* https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-July/014789.html
* http://www.truthcoin.info/images/bmm-outline.txt


Thanks
=========

{{ Thank all the good reviewers }}



Copyright
==========

This document is placed in the public domain.

