    Drivechain Documentation -- Blind Merged Mining
    Paul Sztorc
    October 24, 2017
    Document 3 of 3
    v4.0


Header
=======

    BIP: ????
    Layer: Consensus (soft fork)
    Title: Blind Merged Mining (Consensus layer)
    Author: Paul Sztorc <truthcoin@gmail.com>
            CryptAxe <cryptaxe@gmail.com>
	        Chris Stewart <chris@suredbits.com>
    Comments-Summary: No comments yet.
    Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-???????
    Status: Draft
    Type: Standards Track
    Created: 2017-10-24
    License: BSD-2-Clause

Abstract
==========

Blind Merged Mining (BMM) is a way of mining special extension blocks. It produces strong guarantees that the block is valid, for *any* arbitrary set of rules; and yet it does so without requiring miners to actually do any validation on the block whatsoever.

BMM actually is a process that spans two or more chains. For an explanation of the "whole picture", please see [this post](http://www.truthcoin.info/blog/blind-merged-mining/). This document, however, focuses only on the required modifications to mainchain Bitcoin.

To support BMM, the mainchain is asked to accomplish two goals:
1. Track a set of ordered hashes (the merged-mining).
2. Allow miners to "sell" the act of finding a sidechain block (through the use of a new extended serialization transaction type).

These goals are accomplished by forcing nodes to validate two new messages (M7, M8), and track data in one new database (D3).

Copyright
==========

This BIP is licensed under the BSD 2-clause license.


Motivation
============

Regular "Merged-Mining" (MM) allows miners to reuse their hashing work to secure other chains (for example, as in Namecoin). However, traditional MM has two drawbacks:

1. Miners must run a full node of the other chain. (This is because [while miners can effortlessly create the block] miners will not get paid unless the block that they merge mine is a valid one. Therefore, miners must assemble a valid block first, then merge mine it.)
2. Miners are paid on the other chain, not on the regular BTC mainchain. For example, miners who merge mine Namecoin will earn NMC (and they will need to sell the NMC for BTC, before selling the BTC in order to pay for electricity).

Blind Merged-Mining (BMM) attempts to address both shortcomings.


Specification
============

Note: This document uses the notation side:\* and main:\* in front of otherwise-ambiguous words (such as "block", "node", or "chain").

As stated above, we have two goals: [1] create and monitor an alt-chain (defined only by a deterministic list of hashes), and [2] allow miners to "sell" the act of finding a sidechain block (through the use of a new extended serialization transaction type).

### Sidechain Critical Data ("Sidechain Mini-Header")

Specifically, per side:block per side:chain, we track the following 34 bytes of information:

    1-byte   - ChainIndex ("Account Number" in hashrate-escrows.md, "Sidechain Number (nSidechain) in the code")
    1-byte   - prevBlockRef (an index which points to this side:block's parent side:block)
    32-bytes - sideHeaderHash (also known as "h*" / hashCritical, the hash of the sidechain block)


The **ChainIndex** indicates which sidechain this critical data is relevant to. As we may eventually have more than one sidechain, this serves as an identifier similar to the Bitcoin network's magic bytes (0xF9BEB4D9). Drivechains however only need to use 1 byte for the identifier (there is a hard limit of 256 sidechains identified as 0-255). The **sideHeaderHash** is the hash of a side:block which will receive PoW via BMM. The **prevBlockRef** forms the set of headers into a blockchain structure by making the headers refer to a previous header. It is most similar to Bitcoin's hashPrevBlock.

Where does this data come from, and how does it get around?

#### Creating / Broadcasting This Data

##### Creation

By the time Blind Merged Mining can take place, the ChainIndex is globally known (it is the "Account Number" in D1 [see previous BIP]). Each sidechain, when activated by soft fork, will take one of the 0-255 available indexes.

The other two items, sideHeaderHash and prevBlockRef, are created by sidechain nodes. sideHeaderHash is quite straightforward -- side:nodes build side:blocks, and take the hash of these.

The final item, prevBlockRef, is a little more complicated. It is an integer that counts the number of "skips" one must take in the side:chain ratchet in order to find the side:block's parent block. In practice, this value will almost always be zero. It will only be a value other than zero when invalid sidechain blocks have been mined, or when a side:node intentionally wants to orphan some side:blocks (if a side:node wants to orphan the most-recent N blocks, the value of the current block will be equal to N ; in the block after that it will be back to zero).

![dots-image](https://github.com/drivechain-project/docs/blob/master/images/bmm-dots-examples.png?raw=true)

Since the hashes themselves are already ordered by the mainchain, tracing the blockchain's path by index (prevBlockRef) will be the same as tracing it by identifying a list of hashes. In other words, the ordering given via each side:block's "prevBlockRef" will be isomorphic to an ordering given by each side:block's "prevSideHeaderHash" ... if "prevSideHeaderHash is defined to be the sidechain's equivalent of the mainchain's "prevBlockHash". It will be possible to freely convert from one to the other. See M8 to learn more about how these hashes are requested by sidechain block creators to be included in the mainchain.

Now that we know what our critical data is, and how it is made, how is this data broadcast and stored?

##### Broadcast

Mainchain nodes are going to need this data later, so it must be easy to find. We will put it into the coinbase via OP RETURN.

#### M7 -- "Blind-Mine the Sidechain(s)"

Thus, (for n sidechains) we have a coinbase output with:

    1-byte    - OP_RETURN (0x6a)
    4-bytes   - Identifying flag bytes / message header
    32-bytes  - h*
    5~7-bytes - the rest of the Sidechain Mini-Header

This can be an individual output for each sidechain, {{ or to save some amount of block space we can combine the information for each sidechain into a single scriptPubKey. Doing this means that we wont be duplicating the OP_RETURN prefix or identifying flag bytes. This would take up 9,479 bytes for 256 sidechains assuming that 5 bytes are used for the Critical Data bytes (non h* parts of the Sidechain Mini-Header). Conveniently just below the 10 KB scriptPubKey size limit }}

We are left with, at most, one  (h\*, prevBlockRef) pair per sidechain per block. This data is added directly to D3, a new database.


#### D3 -- "RecentSidechains_DB"

To suit our purposes, the mainchain full nodes will need to keep track of the most recent 4000 (h\*, prevBlockRef) pairs.

Therefore, D3 would look something like this:


           BlockHeight    SC_1    Blks_Atop_1   SC 2    Blks_Atop_2   SC 3    Blks_Atop_3
            ---------    ------   ---------    ------   ---------    ------   ---------
       1.    401,005     (h*, 0)     7985      (h*, 0)        1      (h*, 0)        0
       2.    401,006     (h*, 0)     7984      (h*, 0)        0      (h*, 1)     7801
       3.    401,007     (h*, 0)     7983      (h*, 5)     2027      (h*, 0)        0
       4.    401,008     (h*, 0)     7982      (h*, 0)     2028      (h*, 1)     7800
      ...     ...
    3999.    409,003     (h*, 0)        1      (h*, 0)        0      (h*, 0)        1
    4000.    409,004     (h*, 0)        0      (h*, 1)        0      (h*, 0)        0


When new sidechains (or "hashrate escrows") are soft-forked into existence, a new column is added to D3 (to contain any BMMing that might be done on it).

For each sidechain we also track the field "Blocks Atop". This is the number of side:blocks that are "on top" of the specified side:block. These might be regarded as "side:confirmations" (pseudo-confirmations that are specific to each sidechain).

D3 also contains a column (not shown) for each sidechain containing "prevSideBlockHash". This value is is either derived from prevBlockRef; or else it is given explicitly (in which case it is the converse: prevBlockRef is derived from prevSideBlockHash).

#### Coinbase Cache
To efficiently keep track of the above data, without having to constantly load entire blocks (4,000 of them) from disk, we temporarily cache enough coinbases in the chain index to maintain D3.

### M8 -- Paying miners to include BMM data in their coinbase outputs

This section introduces a new type of transaction, which allows sidechain block creators to request, and pay for, a critical hash to be included in a specific block by mainchain miners. See [the Blind Merged Mining spec](http://www.truthcoin.info/blog/blind-merged-mining/). This txn allows miners to "sell" the act of mining a sidechain block. By taking advantage of this option, miners earn tx fees for mining sidechains...truly "for free". They do not even need to run sidechain nodes, and the tx-fees they earn are in mainchain BTC. As a result, sidechains affect all miners equally and do not affect the mining ecosystem.

M8 will ultimately come in two versions. The second version will be specialized for use in the Lightning Network and must use the full 32-byte prevBlockHash (ironically, this larger transaction is likely to be off-chain and ultimately cheaper for the Bitcoin network to process). The first version of M8, in contrast, cannot be used inside the Lightning Network but only requires a single byte prevBlockRef to maintain chain order. It is important to have both, because some side:nodes may be unwilling or unable to open a payment channel with each main:miner even though eventually it seems that the lightning version will be preferred.

#### Setup

We define **"Mary"** as a mainchain miner, and **"Simon"** as a sidechain node / sidechain block creator.

The goal is to construct a payment from Simon to Mary, such that:

1. If the critical data conditions are met, **Mary** can claim the outputs of the transaction with finality.
2. If the critical data conditions are not met, the outputs become immediately available again to **Simon**.

Simon will use M8 to create a payment (think escrow) where the payment only goes through if the terms Simon has requested are met. For BMM specifically, this means that Simons is entering a contract to pay mainchain miner Mary if and only if the critical hash Simon has requested is published in the coinbase of the exact block that Simon wants this hash committed in. Mary will be able to create an output for herself using Simon's escrowed bitcoin. Although may will likely spend this in the very same block, the output will have a maturity period of 100 blocks on the mainchain as well as at least 100 blocks atop of this critical hash in the ratchet system. See "Forward Progress (The Need for a "Ratchet")" for more details about the ratchet system and blocks atop.

If the conditions are not met, Mary will be unable to claim the money. The transaction will be removed from the mempool (which may end up being a second mempool of its own). This transaction is only valid for a certain block height, immediately after that the transaction becomes invalid. The outputs will be returned to Simons wallet as if they had expired in the mempool after waiting too long (this is relevant specifically to the wallet GUI).

##### Forward Progress (The Need for a "Ratchet")

The "ratchet" concept is an attempt to harmonize incentives among the main and side chain(s). In Bitcoin, miners must meet another condition besides the well known PoW requirement in order to get paid: the block that they find must be followed by 100 child blocks (coinbase maturity). This incentivizes all miners to focus on the global chain health, and work towards building blocks from the chain tip.

We will need to ensure that the sidechain is making "forward progress" as well, without tracking too much about the sidechain such that we burden Bitcoin (see [1] and [2]) all while still allowing the sidechain to reorganize [3].

* [1] https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-July/014789.html
* [2] http://www.drivechain.info/faq/index.html#what-is-the-difference-between-drivechain-and-extension-blocks
* [3] http://www.truthcoin.info/blog/blind-merged-mining/#handling-reorganizations

Bitcoin and sidechains must cooperate (and move both chains forward) to achieve the mutual goal of being paid for their work. Therefore the ratchet system must keep track of sidechain "mini-headers" (see Sidechain Critical Data ("Sidechain Mini-Header")) and count the "blocks atop" maturity of the related side:blocks. By requiring side:block maturity without actually making the mainchain track the side:blocks we encourage progress of both the main and side chains, without placing undue burden on any miner (again, they don't even need to run a sidechain node to profit from this).

Simon's offer to Mary to include a critical hash in exchange for payment must be *atomic*. The "ratchet" concept helps to construct a very tight connection between two things:

1. The sidechain-block-generator "Simon" paying himself the side:block's side:tx-fees (which he receives in 100 sidechain block atop hence).
2. "Simon" making a mainchain main:btc payment to a mainchain miner "Mary".

Either both of the two should succeed, or else both should jointly fail.

However, absent our intervention, there are cases in which [2, the payment to Mary] succeeds but [1, side:tx-fees] fails. One such case is when a side:block contains unusually high side:tx-fees. Here, there will be many requests to include a critical hash in exchange for payment submitted to Mary, but only one can be included in each main:block per sidechain. Without an incentive to make "forward progress", Mary is likely to include one of the highest paying requests in the next main:block (and the main:block after that, and so on). Mary will "blindly" include high-paying requests for *older* blocks, unless something prevents her from doing so.

To address these potential issues, we utilize the concept of "Blocks_Atop" (the "side:confirmations") that we mentioned earlier. As previously mentioned, Mary will not be able to spend Simon's M8 payment until satisfying the blocks atop (side:confirmations) requirement.

#### M8 -- The two forms of M8 transactions

As previously mentioned, M8 can take two forms. The first does not require the Lightning Network, but does require our new transaction type (see above) which has new validation rules including the requirement to be immediately included in the exact block requested, otherwise becoming invalid. The second form inherits immediate expiration from the Lightning Network itself, but requires extra preparation and a different/larger message.

Both forms require that certain Critical Data will be committed to within the coinbase of the block that the
transaction is included in. For the non Lightning version, we have created a new extended serialization transaction type (very similar to how segwit handles witness data (the witness stack)).

##### M8_V1 - Critical Data format with no Lightning Network

Critical data structure for non Lightning use:
    5~7-bytes - (Identifying flag bytes, ChainIndex, prevBlockRef)
    32-bytes  - h* side:block hash

We impose the following validity-rules for these critical data txn(s):

1. The txn's content, when examined, must match part of the main:block's content. Specifically, the (ChainIndex, h\*) pair of the txn, must match one of the (ChainIndex, h\*) pairs in the M7 of this main:block.
2. Only one payment per sidechain per main:block is valid. In other words, if 400 people all try to bm-mine the sidechain with ChainIndex==4, then not only is it the case that only one side_4:block can be found, but it is also the case that only the corresponding M8 txn can be included (out of all of the 400 M8s which are for ChainIndex==4).
3. Simon's txns must only be valid for the current block; afterward, they immediately expire. This is because Simon's intended prevBlockRef & side:block contents will most likely change from one main:block to the next.

To impose new requirements at the transaction level, we make use of the extended serialization transaction format introduced by Segwit. If a dummy vin bit is present, we check which flag has been set. If the flag is set to 2 (0010), the transaction contains Critical Data and requires that our new validation rules be met in order for the txn to be valid in a block. Unless all of the requirements for sidechain critical data transactions are met by the block it is included in, the transaction is invalid. With SegWit, this extra data is the segwit signature stack, and the extra requirements are the signatures' locations and validity. In the sidechain BMM critical data transactions, the extra data is the (ChainIndex, h\*) pair, which must meet the first two requirements (above) as well as the main:blocknumber, which must meet the third requirement (above). Note The main:blocknumber does not take up any additional space compared to a normal txn, as we reuse the locktime field for our purposes.

![extra-data-image](https://github.com/drivechain-project/docs/blob/master/images/witness-vs-critical.png?raw=true)

These types of transactions have slightly different mempool behavior, and should probably be kept in a second mempool. These txns are received, checked immediately, and if valid they are evaluated for inclusion in a block. If they are not able to be included in the specific requested block (if the block height requested has been surpassed by the chain tip), they are discarded. In fact, after any main:block is found, everything in this "second mempool" can be discarded as new payments will be created immediately for the next block height. (This includes cases where the blockchain reorganizes.) There is no re-evaluation of the txns in this mempool ever -- they are evaluated once and then either included or discarded. To be clear, when the transaction is received we are able to evaluate its validity, and do not need to rescan these transactions again, they are simply discarded when we move on to the next block.

These payments (M8) will *always* be directed specifically to miners and nobody else. Therefore, non-mining nodes do not need to keep them in any mempool at all. Non-miner nodes can just wait for a block to be found, and check the txn then. These transactions more resemble a stock market's pit trades (in contrast, regular Bitcoin txns remind me more of paper checks).

##### M8_V2 Critical Data format when using Lightning network

    4-bytes - Identifying flag bytes
    1-byte  - M8 Version (in this case, equal to "2")
    65-byte - 'Big' Sidechain Mini-Header (1+32+32); This consists of: (ChainIndex, h*, prevSideBlockHash)

M8_V2 requires having a LN-channel open with a miner. This may not always be practical (or even possible), especially today.

Notice that, in M8_V1, Simon could reuse the same h\* all he wanted, because only one M8_V1 could be included per main:block per sidechain. However, on the LN no such rule can be enforced, as the goal is to push everything off-chain and include *zero* M8s. So, we will never know what the M8s were or how many had an effect on anything.

Therefore, Simon will need to ensure that he **gives each Mary a different h\***. Simon can easily do this, as he controls the side:block's contents and can simply increment a nonce -- this changes the side:block, and changes its hash (ie, changes h\*).

With a unique h\* per Mary, and at most 1 h\* making it into a block (per sidechain), we can guarantee that only one of the M8_V2's critical data can be committed to in a single main:block. Only one M8_V2 will be valid in any main:block and by giving each miner (who Simon has a payment channel open with) a different h*, Simon can figure out which miner followed through with the commit, and know that only one such commit went through. Furthermore, if this Simon's requested critical data is not found in a block, none of the M8_V2 payments will be spendable by the Mary(s), because none of the h\* in question have ever made it into D3 (which is always on-chain) and no blocks atop will be accumulated.

Deployment
===========

This BIP will be deployed by "version bits" BIP9 with the name "blindmm" and using bit ???.

For Bitcoin mainnet, the BIP9 starttime will be midnight ??????????? UTC (Epoch timestamp ???) and BIP9 timeout will be ??????????? UTC (Epoch timestamp ???).

For Bitcoin testnet, the BIP9 starttime will be midnight ??????????? UTC (Epoch timestamp ???) and BIP9 timeout will be ??????????? UTC (Epoch timestamp ???).

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

Thanks to everyone who contributed to the discussion, especially: ZmnSCPxj, Adam Back, Chris Stewart, Peter Todd, Dan Anderson, Sergio Demian Lerner, Matt Corallo, Sjors Provoost, Tier Nolan, Erik Aronesty, Jason Dreyzehner, Joe Miyamoto.
