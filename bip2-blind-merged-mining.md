Drivechain Documentation
===========================

    Paul Sztorc 
	August 16, 2017
	Document 3 of 3
	v3.0

Note from Paul: Double brackets "{{" and "}}" surround parts that are "unfinished".


Concept Two: Blind Merged-Mining
==================================


Again, this focuses only on the tasks that mainchain Bitcoin must perform.

For a description of functions each sidechain must perform, [this](http://www.truthcoin.info/images/bmm-outline.txt) may be more helpful.

For a description of the "whole picture", [this](http://www.truthcoin.info/blog/blind-merged-mining/) may be most helpful.


## Motivation

Regular "Merged-Mining" (MM) allows miners to reuse their hashing work to secure other chains (for example, as in Namecoin). However, traditional MM has two drawbacks:

1. Miners must run a full node of the other chain. (This is because, while meeting the "difficulty requirement" is free, miners will not get paid unless their block is valid. Therefore, they must assemble a valid block first, then merged-mine it.)
2. Miners are paid on the other chain, not on the regular BTC mainchain. (For example, a miner who merged-mines Namecoin will earn NMC, which he or she will sell for BTC, before selling that BTC in order to pay for electricity.) 

Blind Merged-Mining (BMM) attempts to address those shortcomings.


## Description

The goals are to [1] track a set of ordered hashes (the merged-mining), and [2] allow miners to "sell" the act of finding a sidechain block (through the use of a new OP CODE -- OP CheckBribeVerify).


### Critical Sidechain Info ("Sidechain Mini-Header")

Specifically, we track 35 bytes of information, per side:block per side:chain:

    1-byte   - ChainIndex (known as "Account Number" in hashrate-escrows.md , or as "Sidechain Number")
    32-bytes - sideHeaderHash (also known as "h*", conceptually this is a hashMerkleRoot [or the blockHash, aka prevBlockHash of *this* block] with arbitrarily greater flexibility)
    2-bytes  - prevBlockRef (an index which points to this side:block's parent side:block)
    
The ChainIndex keeps data organized when there are more than one sidechain. The sideHeaderHash determines the data that each side:block is expected to contain. The prevBlockRef forces the set of headers into a blockchain structure.

This involves three new message types (M7, M8, and M9), and a new database (D3).


### Creating / Broadcasting This Data

#### Creation

By the time Blind Merged Mining can take place, the ChainIndex is globally known (it is the "Account Number" in D1 [see previous BIP]).

The other two items, sideHeaderHash and prevBLockRef, are created by sidechain nodes. The first, sideHeaderHash, is quite straightforward -- side:nodes build valid side:block-headers, and take the hash of these. The prevBlocRef value is more complicated. It is an integer that counts the number of "skips" one must take in the side:chain in order to find the current side:block's parent block. In practice, this value will always be zero, except in cases where invalid sidechain blocks have been mined, or when a side:node intentionally wants to orphan some side:blocks (if a side:node wants to orphan the most-recent N blocks, the value of the current block will be equal to N ; in the block after that it will be back to zero).

~ dot examples image ~

As we will see, prevBlockRef will be isomorphic to "prevSideHeaderHash" (this "prevSideHeaderHash is sidechain's equivalent of the mainchain's "prevBlockHash") -- it will be possible to freely convert from one to the other.

The "Bribe" payment, M8, will ultimately come in two versions. The second version is specialized for use in the Lightning Network and must used the full 32-byte prevBlockHash (ironically, this larger transaction is likely to be off-chain and ultimately much cheaper for the Bitcoin network to process). The first version of M8, in contrast, cannot be used inside the Lightning Network but will only require the 2-byte prevBlockRef. (This is important because some side:nodes may be unwilling or unable to open a channel with each main:miner.)

#### Broadcast

Mainchain nodes are going to need this data later, so it must be easy to find. We will put it into the coinbase via OP RETURN.

### M7 -- "Blind-Mine the Sidechain(s)"

Thus, (for n sidechains) we have a coinbase output with:

    1-byte - OP_RETURN (0x6a)
    1-byte - Push the following (4+(n*35)) bytes (0x??)
    4-byte - Message header (0x????????)
    (4+(n*35))-byte - A sequence of bytes, of the three items directly mentioned above ("Sidechain Mini-Header").
	

This data is parsed by laying it in sequential 35-byte chunks (any remaining data --ie, some final chunk that is less than 35 bytes in length-- has no consensus meaning).

Each 35-byte chunk is then parsed to obtain the data outlined above (in "Description"). If two 35-byte chunks being with the same "Sidechain number" (ie, if the two chunks have the same first byte), then only the first chunk has consensus meaning.

We are left with, at most, one  (h\*, prevBlockRef) pair per sidechain per block. This data is added directly to D3, a new database.

	
### D3 -- "RecentSidechains_DB"

To suit our purposes, the mainchain full nodes will need to keep track of the most recent 8000 (h\*, prevBlockRef) pairs.

Therefore, D3 would look something like this: 
       
	   
           BlockHeight  CB_Index    SC_1   Blks_Atop_1   SC 2   Blks_Atop_2   SC 3   Blks_Atop_3      
            ---------    ------    ------   ---------   ------   ---------   ------   ---------
       1.    401,005        2      (h*, 0)     7985     (h*, 0)        1     (h*, 0)        0
       2.    401,006        4      (h*, 0)     7984     (h*, 0)        0     (h*, 1)     7801
       3.    401,007        2      (h*, 0)     7983     (h*, 5)     2027     (h*, 0)        0
       4.    401,008        2      (h*, 0)     7982     (h*, 0)     2028     (h*, 1)     7800
      ...     ...                                                                  )
    7999.    409,003        3      (h*, 0)        1     (h*, 0)        0     (h*, 0)        1
    8000.    409,004        2      (h*, 0)        0     (h*, 1)        0     (h*, 0)        0


When new sidechains are soft-forked into existence, a new column is added to D3 (to contain any BMMing that might be done on it).

D3 also contains a column for each sidechain called "Blocks Atop", which computes how many side:blocks are "on top" of the current side:block. These might be regarded as "pseudo-confirmations" that are specific to each sidechain.

D3 also contains a column (not shown) for each sidechain containing "prevSideBlockHash". This value is is either drived from prevBlockRef; or else it is given explicitly (in which case it is the converse: prevBlockRef is derived from prevSideBlockHash).


### New Validation Rules

As mentioned above, M7s cause data to be added to D3, which is tracked by the mainchain for a period of time.

Specifically, each mainchain node tracks, per sidechain, the last 8000 M7s {{ although, here I have it as the last 8000 main:blocks ... not sure which is better }}. In addition, node software must update the BlocksAtop value for all the Row's entries. In practice this is quite easy, because one merely needs to set the new entry to a BlocksAtop value of "0", and then scroll backwards through all of the prevSideBlockHash es and add 1 to everything in this 'chain'. {{possibly an index column could make this even easier}} 


## M8 and M9 -- Arranging the Payments

This section introduces a new type of transaction, the "bribe" of [the Blind Merged Mining spec](http://www.truthcoin.info/blog/blind-merged-mining/). This txn allows miners to "sell" the act of mining a sidechain block. By taking advantage of this option, miners earn tx fees for mining sidechains...truly "for free". They do not even need to run sidechain nodes, and the tx-fees they earn are in mainchain BTC. As a result, sidechains affect all miners equally and do not upset the mining ecosystem.

### Setup

We define **"Mary"** as a mainchain miner, and **"Simon"** as a sidechain node.

The goal is to construct a payment from Simon to Mary:

1. If *'conditions_1'* are met, **Mary** can claim the money with finality.
2. If, instead, the *'timeout_condition'* is met, **Simon** can reclaim the money.

Note: see https://lightning.network/ for info about how this is often done.

Simon will use M8 to pay into a "Bribe Escrow" txn. Mary will later withdraw the money from M8 using with M9. M9 will meet the requirements of our new OP code.

( Else, Mary will be unable to claim the money, and after the timeout condition is met, it will be exactly as if Simon has merely made a payment to himself. He will ultimately be able to regain access to the funds, this txn might be regarded as an "M9" but at this point it makes no practical difference. )


### Goals (this is rather philosophical, and skippable)

#### Immediate Expiration ("Fill-or-Kill")

We would like to make special guarantees to the counterparties of this transaction. Specifically, instead of Simon's "payments" to Mary, we prefer that Simon gives Mary an "offer" to either accept or decline.

Crucially, we want Simon to safely make many offers to several different Mary's, in realtime (ie, quickly and off-chain). If Simon's offers will *immediately expire*, then Simon will feel comfortable making offers all day long. This fact means that the Marys collectively gain access to a large set of offers. 


#### Forward Progress (The Need for a "Ratchet")

<!--  Too Long!

The "ratchet" concept is an attempt to harmonize incentives among the main and side chains. In regular mainchain Bitcoin, miners are paid to find blocks, as is commonly known. However, less commonly known is that miners must meet another condition in order to get paid: the block that they find must be followed by 100 blocks. This helps keep miners focused on the global chain health, in addition to their local obsession with finding the next block.

We will need to ensure that the sidechain is making "forward progress", without tracking too much about the sidechain (see [1] and [2]) and while still allowing the sidechain to reorganize [3].

* [1] https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-July/014789.html
* [2] http://www.drivechain.info/faq/index.html#what-is-the-difference-between-drivechain-and-extension-blocks
* [3] http://www.truthcoin.info/blog/blind-merged-mining/#handling-reorganizations


Note: The two chains (main, and side) cooperate to achieve this mutual goal. While the mainchain enforces the new rules described here, each sidechain enforces its own rules (not described here). However, we plan on each sidechain using two rules in particular: first, the sidechain [a] knows exactly where to look for these hashes (and, therefore, will always be able to find one unique has per mainchain block) and the sidechain also [b] accepts this hash as being equal to "a prevBlockHash which meets the [side]chain's difficulty target". Second, the sidechain [a] knows exactly where to look for the prevBlockRef, and [b] interprets the main:chain-sourced prevBlockRef to actually define the side:block's side:parentblock.
-->

Simon's bribe offer must be *atomic*. The "ratchet" concept helps to construct a very tight connection between two things:

1. The sidechain-block-generator "Simon" paying himself the side:block's side:tx-fees (which he receives in 100 sidechain blocks hence).
2. "Simon" making a mainchain main:btc payment to a mainchain miner "Mary".

Either both of the two should succeed, or else both should jointly fail.

But, without a ratchet-concept, there are cases in which the second [bribe] succeeds but the first [side:tx-fees] fails. One such case is when a side:block contains unusually high side:tx-fees. Here, there will be many candidate bribes submitted to Mary, but only one can be included in each main:block. Without an incentive to make "forward progress", Mary is likely to include one of these large bribes in the next main:block (and the main:block after that, and so on). Mary will "blindly" include high-paying bribes for *older* blocks, unless something prevents her from doing so.

To address these, we have the concept of "Blocks_Atop" (the "pseudo-confirmations"). Our new OP Code will refuse to main:pay Mary until the side:block in question is buried by X=100 child side:blocks. At this point, both Mary and Simon will get their money at the same time.


### M8 -- Pay to Bribe Escrow (The "Bribe")

M8 pays from Simon into a "Bribe Escrow". (M9 withdraws from this escrow.)

As previously mentioned, M8 can take two forms. The first does not require the Lightning Network, but has new requirements for Immediate Expiration (see above). The second inherets Immediate Expiration from the Lightning Network itself, but requires extra preparation and a different/larger message.

#### M8_V1 - No Lightning Network

We need a way for Simon to constuct many payments to multiple Marys, such that only one of these is ever included; and only then if Simon's txn is expected to coincide with the finding of Simon's side:block.

We do this by imposing validity rules on the txn itself.

The bribe txn is only valid if:

* The bribe is for a side:block that was BMMed in this main:block. Ie, the txn's (ChainIndex, h\*) pair matches a (ChainIndex, h\*) pair in a M7 message of this block.
* Because Simon's intended prevBlockRef may change from one main:block to the next, Simon's txns must only be valid for the current block; afterward they immediately expire.

To impose new requirements on the transaction level (not the block level nor the TxOutput level), we will need a new Transaction Version (version 3).

~ tx version 3 diagram ~

It is as follows:

    1-byte -- Transaction Version (equal to the value of "3" in this case)

    35-byte -- The Sidechain's "Mini-Header" (see above). The ChainIndex, h*, and prevBlockRef.
    2-bytes -- mainAnchor (the main:blocknumber in which this txn can be included ; in practice it is the blocknumber modulo 65536 because there is no need to take up more than 2-bytes).

    ?-byte -- TxIn (the same as in earlier txn versions, Simon's source of funds)
    ?-byte -- TxOut (the same as in earlier txn versions, Mary's destination info [locked to OP_CBV] and Simon's Change)

This txn structure conserves main:blockspace, because it is the easiest way to refer to a previous sidechain block in 4 bytes, (prevBlockRef + FoK_nLockTime) instead of 32 (prevBlockHash).

These types of transactions have slightly different mempool behavior, and will probably be need to be kept in a second mempool. These txns are received, checked immediately, and if valid they are evaluated for inclusion in a block. If they are not included in the current block, they are discarded. In fact, after a block is found, everything in this "second mempool" should be discarded immediately. (This includes cases where the blockchain reorganizes.) There is no re-evaluation of the txns in this mempool -- they are constantly in a state of being discarded.

We will discuss M9 next (both versions, as they are very similar), before discussing M8_V2.

### M9 -- OP CheckBribeVerify

#### M9_V1 -- No Lightning Network

When Simon constructs M8_V1, one of the TxOutputs is a payment to Mary, and this payment is locked with a new OP Code, "OP CheckBribeVerify" (CBV).

OP_CBV behaves very similarly in V1 and V2, but not identically. In V1, the contents are:

    1-byte -- OP CheckBribeVerify
    1-byte -- Bribe Version (in this case, equal to "1")
    35-byte -- 'Regular' Sidechain Mini-Header (1+32+2); ChainIndex, h*, prevBlockRef 

#### M9_V2 -- Within Lightning Network

In V2, OP_CBV contains:

    1-byte -- OP CheckBribeVerify
    1-byte -- Bribe Version (in this case, equal to "2")
    35-byte -- 'Big' Sidechain Mini-Header (1+32+32); ChainIndex, h*, prevSideBlockHash

In both cases, M9 can only be included in a main:block (ie, can only avoid evaluating to a FALSE script outcome) if [1] the Sidechain Mini-Header info is currently present in D3 (of this main:block) and [2] the relevant entry in D3 has a "Blocks_Atop" value of at least 100.


### M8 Again (Lightning Network)

A second version of M8 emerges naturally from M9_V2 along with the Lightning Network. However, this requires having a LN-channel open with a miner. This may not always be practical (or even possible), especially today.


#### M8_V2

In M8_V1, Simon could reuse the same h\* all he wanted, because only one M8_V1 can be included per block per sidechain. However, on the LN no such rule can be enforced, as the goal is to push everything off-chain and include *zero* M8s.

Therefore, Simon will need to ensure that he **gives each Mary a different h\***. Simon can easily do this, as he controls the side:block's contents and can simply increment a nonce.

With a unique h\* per Mary, we can guarantee that M9_V2 will always fail for at least m-1 Marys (if there are m total). If this Simon's block is not found, M9_V2 will fail for all Marys, as the h\* in question will never make it into D3 (let alone have 100 side:blocks built on top of it).

Thus, consider an example where: Simon starts with 13 BTC, Mary starts with 40 BTC, the side:block's tx-fees currently total 7.1 BTC, and Simon is keeping 0.1 BTC for himself and paying 7 BTC via bribe.

We start with (I):    

    Simon 13 in, Mary 40 in ; 53 in total
        Simon's version [signed by Mary]
            13 ; to Simon if TimeLock=over; OR to Mary if SimonSig
            40 ; to Mary
        Mary's version [signed by Simon]
            40 ; to me if TimeLock=over; OR to Simon if MarySig
            13 ; to Simon    


And both parties move, from there to "M8_V2" (II):    

    Simon 13 in, Mary 40 in ; 53 in total
        Simon's version [signed by Mary]
            6 ; to Simon if TimeLock=over; OR to Mary if SimonSig
            40 ; to Mary
            7 ; to Mary if CheckBribeVerify(bI,h*,pSBH); OR to Simon if LongTimeLock=over
        Mary's version [signed by Simon]
            40 ; to Mary if TimeLock=over; OR to Simon if MarySig
            6 ; to Simon
            7 ; to Mary if CheckBribeVerify(bI,h*,pSBH); OR to Simon if LongTimeLock=over

From here, if the h\* side:block in question is BMMed, they can proceed to (III):

    Simon 13 in, Mary 40 in ; 53 in total
        Simon's version [signed by Mary]
            6 ; to Simon if TimeLock=over; OR to Mary if SimonSig
            47 ; to Mary
        Mary's version [signed by Simon]
            47 ; to me if TimeLock=over; OR to Simon if MarySig
            6 ; to Simon   

Although, if Simon proceeds immediately, he removes the protection of the 'ratchet'. Ie, Simon removes Mary's incentive to care about blocks being built on this side:block. If Simon's side:block is orphaned, he loses his 7 BTC. Simon can either play it safe, and wait the full 100 side:blocks before moving on (to the third LN txn, above); or else Simon can take the risk if he feels comfortable with it.

If the h\* side:block is not found, then (II) and (III) are basically equivalent to each other. Simon and Mary could jointly reconstruct I and go back there, or proceed from there to a new version of II with a different h\*.

<!-- obsolete

#### Notes / Improvements

* To discourage Mary from locking up Simon's money in an inconvenient way, we could force Mary to also send a very small amount of her money into the transaction -- much less than she is being paid, but enough to reimburse Simon for the inconvenience of having his money locked away temporarily. This has the added benefit of making Simon care less about having his money unfairly locked up -- we can increase the timeout period from 500 main:blocks to a larger number.
* As usual, Simon and Mary can utilize the lightning network, such that their txns never needs to make it onto the chain, unless there is a dispute.


-->



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
