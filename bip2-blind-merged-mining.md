Drivechain Documentation
===========================

    Paul Sztorc 
	August 16, 2017
	Document 3 of 3
	v0.2

Note from Paul: Double brackets "{{" and "}}" surround parts that are "unfinished".


Concept Two: Blind Merged-Mining
==================================


Again, this focuses only on the tasks that mainchain Bitcoin must perform.

For a description of functions each sidechain must perform, [this](http://www.truthcoin.info/images/bmm-outline.txt) may be more helpful.

For a description of the "whole picture", [this](http://www.truthcoin.info/blog/blind-merged-mining/) may be most helpful.


## Motivation

Merged-Mining (MM) allows miners to reuse their hashing work to secure other chains (for example, as in Namecoin). However, MM has two drawbacks:

1. Miners must run a full node of the other chain (because they need to assemble a valid block to work on).
2. Miners are paid on the other chain, not on the regular BTC mainchain.

Blind Merged-Mining (BMM) attempts to address those shortcomings.


## Description

The goals are to [1] track a set of ordered hashes (the merged-mining), and allow miners to "sell" the act of finding a sidechain block (through the use of a new OP CODE -- OP CheckBribeVerify).

Specifically, we track 35 bytes of information, per sidechain:

    1-byte   - Sidechain Number (known as "Account Number" in hashrate-escrows.md)
    32-bytes - prevSideBlockHash (also known as "h*")
    2-bytes  - blockMod (a counter -- in pratice, it is the sidechain's block number modulus 65535)
	

This involves new message types (M7, M8), and a new database (D3).


### Creating / Broadcasting This Data

#### Creation

The sidechain number is globally known.

The other two items are created by sidechain nodes -- first, these nodes build valid sidechain block-headers, and then can then compute the hash and block number (and therefore, the blockMod) of these.


#### Broadcast

Since mainchain nodes are going to rely on this data later, it must be easy to find.

### M7 -- "Blind-Mine Sidechains"

Thus, (for n sidechains) we place data into the coinbase via OP RETURN:

    1-byte - OP_RETURN (0x6a)
    1-byte - Push the following (4+(n*35)) bytes (0x??)
    4-byte - Message header (0x????????)
    (4+(n*35))-byte - A sequence of bytes, of the three items directly mentioned above ("Description").
	

This data is parsed by laying it in sequential 35-byte chunks (any remaining data --ie, a final chunk that is less than 35 bytes in length-- has no consensus meaning).

Each 35-byte chunk is then parsed to obtain the data outlined above (in "Description"). If two 35-byte chunks being with the same "Sidechain number" (ie, if the two chunks have the same first byte), then only the first chunk has consensus meaning.

We are left with, at most, one  (h*, blockMod) pair per sidechain per block. This data is added directly to D3, a new database.

	
### D3 -- "RecentSidechains_DB"

To suit our purposes, the mainchain full nodes will need to keep track of the most recent 4000 (h*, blockMod) pairs.

Therefore, D3 would look something like this: 
       
	   
              SC 1          SC 2           SC 3      
             -------      --------       --------
       1.   (h*, 746)    (h*,   25)     (h*, 1156)
       2.   (h*, 747)    (h*,   26)     (h*, 1157)
       3.   (h*, 748)    (h*,   27)     (h*,  903)
       4.   (h*, 749)    (h*,   28)     (h*,  904)
      ...
    3999.   (h*, 744)    (h*, 4023)     (h*, 3487)
    4000.   (h*, 745)    (h*, 4024)     (h*, 3488)


In addition to the (h*, blockMod) pairs (which are shown), the database D3 also cointains (not shown) the information required to prove that each pair was included in a coinbase in a recent block.

This would consist of (for the most recent X=4000 blocks):

    1. Reusable data (common to all sidechains)
        a. blockNum      -- This (mainchain) block number.
        b. qSidechains   -- Total quantity of sidechains in existence, at this time.
        c. "Linking Data" (How the h*s are located)
	    i.  merklePath   -- The path from the mainchain blockheader's hashMerkleRoot down to its coinbase txn.
	    ii. coinbase txn -- The contents of the coinbase txn itself.
    2. Sidechain specific (data that exists per sidechain):
        a. "chunk" -- location of (h*, blockMod) pair within the coinbase (ie, "1st chunk", "2nd chunk")
	b. "pair"  -- the data itself
	    i.   h* (prevSideBlockHash)
	    ii.  blockMod
	
{{ We could save 1 byte per sidechain per block, by forcing them to take an order. If a sidechain were not mined in this block, it would need to have a dummy placeholder value of 000000000s, which seems awkward in this case (and would waste 32 bytes). So I'm against it for now. }}



## Using This Data


### The Need for a "Ratchet"

The "ratchet" concept is an attempt to harmonize incentives among the main and side chains. In regular mainchain Bitcoin, miners are paid to find blocks, as is commonly known. However, less commonly known is that miners must meet another condition in order to get paid: the block that they find must be followed by 100 blocks. This helps keep miners focused on the global chain health, in addition to their local obsession with finding the next block.

We will need to ensure that the sidechain is making "forward progress", without tracking too much about the sidechain (see [1] and [2]) and while still allowing the sidechain to reorganize [3].

[1] https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-July/014789.html
[2] http://www.drivechain.info/faq/index.html#what-is-the-difference-between-drivechain-and-extension-blocks
[3] http://www.truthcoin.info/blog/blind-merged-mining/#handling-reorganizations


Note: The two chains (main, and side) cooperate to achieve this mutual goal. The mainchain enforces the rules here. Each sidechain enforces its own rules. First, the sidechain [a] knows exactly where to look for these hashes (and, therefore, will always be able to find one unique has per mainchain block) and the sidechain also [b] accepts this hash as being equal to "a prevBlockHash which meets the [side]chain's difficulty target". Secondly, the sidechain [a] knows exactly where to look for the blockMod, and [b] requires the [mainchain-sourced] given blockMod to actually equal the [sidechain-sourced] modulus of their sidechain.


### New Validation Rules

As mentioned above, M7s cause data to be added to D3, which is tracked by the mainchain for a period of time.

Specifically, each mainchain node tracks, per sidechain, the last 4000 blockMods.

A new rule is required:

* Each *new* block can only include an M7, if the blockMod which it inserts (into D3) is within (-4000, +1) of the most recent blockMod avaliable.




### OP CheckBribeVerify

#### Purpose

This section reappropriates an OP NOP, so as to allow miners to "sell" the act of mining a sidechain block. This allows all miners to benefit from sidechains equally -- even those which are unable to run burdensome sidechain nodes.

#### Setup

We define **"Mary"** as a mainchain miner, and **"Simon"** as a sidechain node.

The goal is to construct a payment from Simon to Mary:

1. If *'conditions_1'* are met, **Mary** can claim the money with finality.
2. If, instead, the *'timeout_condition'* is met, **Simon** can reclaim the money.

Note: see https://lightning.network/ for info about how this is often done.

#### Structure

The TxOut in question would include three arguments: (h*, this_blocknumber, timeout).

#### Mary vs Simon

**Mary** would be able to spend it if she can prove (in the main:chain) two things:

1. ..that a side:block of hash h* was mined recently (aka, that h* was included in a main:coinbase recently), AND
2. ..that the side:block has been buried by 200 side:blocks on the side:chain.

The first is proved by using the linking data to show that a (h*,bM) pair exists in D3. The second is shown by [a] showing that a (h*,bM) pair exists in {{the previous}} main:block (ie, one main:block ago), and [b] showing that the two bM values ("bM_new", from one main:block ago ; and "bM_old", the bM from the pair referenced in the previous sentence) obey the following equation: 

* ~~ bM_new > bM_old + 200 [wrong, but instructive] ~~
* bM_new > (  (bM_old + 200) mod 65535  )

In contrast, **Simon** is able to spend the txn (and recapture his money) if Mary goes some length of time (say, 500 main:blocks) without accomplishing what she said she would be able to accomplish.

#### Notes / Improvements

* To discourage Mary from locking up Simon's money in an inconvenient way, we could force Mary to also send a very small amount of her money into the transaction -- much less than she is being paid, but enough to reimburse Simon for the inconvenience of having his money locked away temporarily. This has the added benefit of making Simon care less about having his money unfairly locked up -- we can increase the timeout period from 500 main:blocks to a larger number.
* As usual, Simon and Mary can utilize the lightning network, such that their txns never needs to make it onto the chain, unless there is a dispute.





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

