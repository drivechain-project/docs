Drivechain Documentation
===========================

    Paul Sztorc 
	August 15, 2017
	Document 2 of 3

Note from Paul: Double brackets "{{" and "}}" surround parts that are "unfinished".


Concept One: Hashrate Escrow
=============================

## Summary

One way to think of this is as a 2-of-3 escrow, where the 3rd party (who will arbitrate any disputes) is the set of Bitcoin Miners. In practice these are likely to be "asymmetric sidechains" of Bitcoin (such as Rootstock) or "virtual chains" within Bitcoin (such as proposed by Blockstack in mid-2016).

"Hashrate Escrow" is a clearer way of expressing the (completely-equivalent) concept of "locked to an SPV Proof" which is itself an elaboration of the concept of a "sidechain" as described in a famous paper [1] written partially by some Blockstream co-founders.

[1] https://blockstream.com/sidechains.pdf

This concept consists of new databases, and new message-interpretations.

#### New Databases

* D1. "Escrow_DB" -- a database of "accounts" and their attributes.
* D2. "Withdrawal_DB" -- a database of pending withdrawals from these accounts, and their statues.

#### New Messages

* M1. "Propose New Escrow"
* M2. "ACK Escrow Proposal"
* M3. "Propose Withdrawal"
* M4. (implied) "ACK Withdrawal"
* M5. "Execute Deposit"   -- a transfer of BTC from-main-to-side
* M6. "Execute Withdrawal" -- a transfer of BTC from-side-to-main


#### On the Resource Requirments of New Databases

To insert a new entry into either database (D1 or D2), one needs to first place the new database entry info into a coinbase txn. In other words, the databases are simply reinterpretations of data contained in coinbase txns. The data in question happen to be Messages M1 through M3, all of which are located in the coinbase txn. M5 and M6 are located with the other Bitcoin txns -- namely, inside hashMerkleRoot. With the exception of M4 (see M4 section, below), we just rearrange what is already there.

Because of this, even though "new databases" are created and stored in memory, the previous bandwidth and storage limits are fully respected. (Again, with the exception of M4.)


## Adding Sidechains and Tracking Them (D1, M1, M2)

### D1 -- "Escrow_DB"

The list below enumerates the database fields, their size in bytes, and (optionally) some notes.

1. Escrow Number (1) -- aka "Sidechain Number"
2. Escrow Name/Description (120)
3. Critical Private Key (32) -- This is the Double-SHA256 of the binary release of the reference software
4. Critical Address (25) -- derived from #3
5. Waiting Period (2) -- timing parameter, expressed in "number of blocks", unsigned (ie, max of 65535, or just over 15 months)
6. Voting Period (2) -- (same as #5)
7. Threshold Given (1) -- expressed as a "percentage", but "per 256" (instead of per 100)
8. Threshold Calc (2) -- derived from #6 and #7, equal to ceiling( [#6]*([#7]/256) )
9. Active (1) -- a counter, removes the account from D1 if activation conditions are not met
10. Critical TxID-Index Pair "CTIP" (32,4) -- a variable for keeping track of where the "account balance" is, and keeping all of the BTC in a single member of the UTXO set {{can we trim index from this somehow?}}

Total Bytes needed = 1+120+32+25+2+2+1+2+1+32+4 = 222
(of which, 25+2+1+32+4 = 64 are redundant or self-derived, 158 are critical)

D1 is sorted by field #1 "Escrow Number" (and, as there is only ever one entry per escrow number, this sorting always unique -- if a new entry is added with an existing number, it overwrites the old entry and destroys it).

(The following messages were modeled on SegWit -- https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#commitment-structure )


### M1 -- "Propose New Sidechain"

    1-byte - OP_RETURN (0x6a)
    1-byte - Push the following 162 bytes (0xA2)
    4-byte - Commitment header (0x????????)
    158-byte - the critical bytes mentioned above (fields #1, #2, #3, #5, #6, and #7, to populate a new D1 entry)

* Broadcasting a properly-formatted M1 will get an entry into D1, in the very next block.
* Field #4 will be calculated as per version 1 Bitcoin addresses (see [1]), but with a prefix of "4" instead of "1".
* Field #8 will be derived from #6 and #7 using math.
* The initial values of Fields #9 and #10 are set to zeros.

[1] https://en.bitcoin.it/wiki/Technical_background_of_version_1_Bitcoin_addresses

The account will "reuse" the same address over and over. But notice that there is no privacy loss due to address reuse, because the addresses in question does not "belong" to any specific individual. The transactions could be trivially linked by just watching the account for deposits ...this, in fact, is exactly what the sidechain is doing. Therefore there is no marginal reduction in privacy due to address reuse.


### M2 -- "ACK Sidechain Proposal"

    1-byte - OP_RETURN (0x6a)
    1-byte - Push the following 29 bytes (0x1D)
    4-byte - Commitment header (0x????????)
    25-byte - Commitment hash: RIPMD-160 of Sha256 of a given M1 (above)


* An entry will be removed from D1, unless that block contains an M2 supporting it (this will increment field #9), or the entry has accumulated 256 M2s.
* Therefore, each entry needs to have 256 *consecutive* M2s supporting it, else it will be removed from D1 and must start over. This requires all miners to agree to "soft fork" the sidechain into existence. This makes it easier to curate the list of active sidechains, and prevents destructive interference among contradictory sidechains.
* Once the entry has 256 M2s, then a miner can make a 0 btc payment to the Critical Address, thus activating the chain. This payment becomes the first "critical TxID index pair" (or "CTIP").



## Adding Withdrawal-Attempts and Tracking Their Success (D2, M3, M4)

### D2 -- "Withdrawal_DB"

The list below enumerates the database fields, their size in bytes, and (optionally) some notes.

1. Escrow Number (1)
2. WT^ (32) -- this is a "blinded transaction id" (ie, the double-Sha256 of a "withdrawal" txn that has had two fields zeroed out, see M6)
3. ACKs (2) -- the current total number of "votes", this starts at 0 during the waiting period
4. Age (3) -- total length of time, in blocks, that this WT^ has been inside of D2
5. Waiting Period (2) -- time this entry must sit idle, before accumulating ACKs, pulled from D1's field #5
6. Max Age (3) -- equal to the value (D1's field #5 + D1's field #6)
5. Threshold (2) -- total ACKs needed, pulled from D1's field #8

Total Bytes needed = 1+32+2+2+2 = 39
(of which, 2+2+2 = 6 are redundant or self-derived, 33 are critical)

D2 is always sorted first by field #1 (Escrow Number) and second by field #4 (Age). This imposes a unique sort.

* From one block to the next, "Age" always equals Age+1.
* Removal criteria:
* * When "Age" = "MaxAge", the entry is deleted from the database in the next block.
* * Entries are also deleted in the next block, if the block contains a txn whos txID matches WT^.


### M3 -- "Propose Withdrawal"

    1-byte - OP_RETURN (0x6a)
    1-byte - Push the following 37 bytes (0x25)
    4-byte - Commitment header (0x????????)
    33-byte - the critical bytes mentioned above (fields #1 and #2, to populate a new D2 entry)

* New entires join D2 in the very *next* block after this one. The starting values of fields #3 and #4 are zero, and #5 is pulled over by extracting the relevant value from D1.
* Can only propose one withdrawal per sidechain per block.


### M4 -- "ACK Withdrawal"

M4 is exceptional (in comparison to the other M's) in a few ways. First, its content is not stored anywhere, only the *hash* of its *effect* is stored (in a leaf of a merkle tree who's root is inserted into a mainchain coinbase). M4 alters the contents of D2 -- the *contents* of D2 are consensus critical, but M4 (the process by which nodes reach a new valid D2) can be anything.

In fact, M4 can also be *nothing*. In other words, it may be optional. This is precisely because, from one block to the next, D2 is only allowed to change in a few ways. Therefore, the exhaustive set of "candidate D2s" can be precomputed by full nodes in advance.

If there are n Sidechains and m Withdrawals-per-sidechain\*, then there are (m+2)^n total candidates for the next D2. This is because (per block per sidechain) one of three things can happen: (1) one of the m withdrawal-candidates can be "ACK"ed (or "upvoted" or "promoted"), which automatically downvotes the others; or (2) all withdrawal-candidates can be downvoted, or finally (3) the miners can abstain from voting altogether, leaving the tallies the same.

First, or nodes which validate all sidechains, this simplifies to 2^n -- these nodes only have to choose between the single honest choice or an abstention. Secondly, nodes might make a simplifying assumption: whichever withdrawal was most recently added/upvoted, will be the one which is upvoted next. This simplifies to 3^n candidates.

If the "D2 update" cannot be guess, it must be transmitted in some way. 

Two examples are below:

"Short Form" (Assumes there are no more than 254 active withdrawal-attempts per account)

    4-byte - Commitment header (0x????????)
    1-byte - Version of this message
    N-byte - N is the total number of active accounts ("sidechains"), each byte specifies the position of the single WT that was "upvoted". A value of 0 indicates "downvote everything", a value of 255 indicates abstention.

"Long Form" (Makes no assumptions about anything)

    4-byte - Commitment header (0x????????)
    1-byte - Version of this message
    1-byte - Length (in bytes) of this message; total number of withdrawal attempts; y = ceiling( sum_i(m_i +2)/8 ).
    Y-byte - stream of bits, with a 1 indicating the position of the chosen action [abstain, downvote all, upvote1, upvote2, ...]


If the message is very very large, then nodes may see no reason to broadcast it. This opens up an "exhaustion attack"\*\*. Fortunately even for 200 sidechains and 1,000 withdrawal-attempts each (absurdly unrealistic), the long form M4 is just over 25 KB. (Meanwhile, to initiate this attack, the attacker must give up about (200*990*(1+1+4+33))= 7722 KB in regular blockspace.)


#### D2 Rules (and, by implication, M4 Rules)

From one block to the next, D2 can only be edited in a few strict ways:

* Entries can only be added/removed from D2 if they meet the criteria above (in M3, and implicitly M1 and M2).
* The ACK-counter of any individual entry can only change by (-1,0,+1) relative to its previous entry. {{Should D2 contain a field "previous ACK value"? Would that help with this?}}
* If "Age" < "Waiting Period", then the ACK counter must remain at zero.
* Within a sidechain group, upvoting one withdrawal (ACK=ACK+1) requires you to downvote all other withdrawals in that group. However, the minimum ACK value is zero.

#### Footnotes for M4

\* This represents the worst-case scenario is one where all the Withdrawals are spread evenly over each Sidechain. Under normal operations, there is no reason to expect the all sidechains will have the same number of withdrawals at any given time. In fact, under normal operations, the very *concept* of counting the withdrawals-per-sidechain should be a purposeless one, because there should only be *one* withdrawal at a time. Nonetheless we consider the worst case scenario here.

\*\* Guessing becomes more computationally intensive in a highly adversarial situation where the "limited range" is intentionally expanded. In such a scenario, [a] there are many sidechains, and [b] miners voluntarily sacrifice their scarce block-space by creating a high number of (mutually-exclusive, and hence ultimately invalid) withdrawal attempts and putting these into coinbase transactions; and then agree to all [c] vote on these randomly (guaranteeing that all withdrawals fail, including any true withdrawals) and [d] successfully withhold their random voting strategies from nodes (even including spy-miner-nodes). Under this bizarre scenario, nodes may require computing resources which increase near-exponentially with the number of withdrawals, and it may take a long time for an ignorant node to exhaustively work out the underlying state of Withdrawal_DB. In this case, nodes may need to temporarily stop validating such transactions (as if they had not yet upgraded to support this soft fork). This limitation is not considered to be qualitatively different from other block-publication-incentive considerations. Hence, any attacker with the resources and interests listed above, would certainly not go about attacking in this way (in the author's opinion).




{{ 

Ideas to reduce bandwidth requirements.

Make the guessing very easy:

* Get rid of the idea of abstaining / downvoting -- slightly problematic.

Don't guess:

* Force coinbase txns to contain votes.

}}



## Depositing and Withdrawing (M5, M6)



Both M5 and M6 are regular Bitcoin txns. They are identified by meeting an important criteria: they select a one of the Critical TxID-index Pairs (a "CTIP") as one of their inputs.

Such txns are identified in {{TODO: Link to exact line of Code}}, and are forced to obey two additional criteria:

1. They must contain an output paying "to" the Critical Address [probably in TxOut0]. {{Note about avoiding OP_CheckSig with OP True -- can't remember the status of this}}
2. They must be accompanied by an update to this sidechain's Critical TxID-index Pair (CTIP). The new CTIP must be "this" txn itself.

{TODO: Should be ban the "mixing of sidechains" -- cannot select any other Critical TxID as one of their inputs. This rule may not be necessary.}

{TODO: These criteria are enforced in {{this part}} of {{validation.cpp}} }

The purpose of this is to have all of the escrow's money (ie all of the sidechain's money) in one TxID, so that depositors immediately undo any UTXO bloat they may cause. This simplifies the withdrawal process, as there is no need to worry about cleaning up "dust deposits" (which results in headaches, for example a withdrawal-txn being larger than 1MB, or which can only withdraw some arbitrarily capped amount). Notice that this can be done without loss, as (unless we assume that an account will last forever) all utxos which are deposited must eventually be withdrawn by someone.

The deposits ("M5") are distinguished from the withdrawals ("M6") by simply checking to see if money is "going in", or "out". In other words, we compare the BTC value of the original CTIP to that of new CTIP. If original <= new it is a deposit, if original > new then it is a withdrawal.


### M5. "Make a Deposit" -- a transfer of BTC from-main-to-side

As far as mainchain consensus is concerned, there are no additional criteria.

However, in practice there *are* criteria...specified by the escrow account, (ie the "sidechain" or "virtual chain"). The sidechain is free to invent any way to credit depositor's money, so this message is fully customizable.

One method, is for depositors to append a zero-value OP Return to a Deposit txn, so that the sidechain knows where to credit funds on the sidechain network. Mainchain users must upgrade their wallet software, of course, (on an individual basis) in order to become aware of and take advantage of new deposit-methods.

The requirement that each hashrate escrow be linked to a single TxID does create an interesting inconvenience for depositors. If a user is slow to sign a txn after constructing it (perhaps because the user employs an air-gapped computer, etc), then the signed txn may no longer be valid. This is because the input it selects, may no longer be the Critical TxID (as "the" Critical TxID changes with each deposit). As a result, the transaction must fail, and the user would need to be prompted to remake and resign the txn. If this is problem is too frustrating, users can always make main-to-side transfers using atomic cross chain swaps (or, the LN, if they already have a channel open on both chains).

Fortunately, it is already a part of mainchain consensus that no two txns can spend the same TxID. The only new issue here is the confusion it might create for the user (hence the need for error messages and alternative deposit-methods).


### M6. "Execute Withdrawal" -- a transfer of BTC from-side-to-main

We come, finally, to the critical matter: where users can take their money *out* of the escrow account, and return it to the "regular" UTXO set. As previously mentioned, this txn is one which (a) spends from a CTIP and (b) reduces the quantity of BTC in an account's CTIP. Most of the work has already been done by D1, M3, M4, and D2.

We merely clarify a little further:

* In each block, an entry in D2 is considered an "approved candidate" if the "ACKs" value is above the "Threshold" value.
* A "blinded TxID" is way of hashing the txn in which we first overwrite some parts of the txn with zeros. Specifically, the first 36 bytes of TxIn0 (TxOutHash + TxOutIndex), as well as the first 8 bytes of TxOut0.

Blinding is necessary because of our restriction of the account to a single UTXO-member. Specifically, miners will need to ACK ("vote on") a withdrawal-txn, but during the ACKing process two features of that withdrawal-txn may change: the CTIP (ie, the "way of finding" the UTXO-member), and the total quantity of BTC stored in the account (which will arbitrarily increase with each new deposit). In other words, a withdrawal-attempt is created via M3, but this takes place many blocks before the withdrawal is actually included via M6. During this time, a single new deposit to the account would change its CTIP and its value. So, we will link the txn to its CTIP using D1; and we will force *all* of the un-withdrawn BTC to be sent back, by forcing sum(input_values) to equal sum(output_values. The latter requirement eliminates the possibility of including a txn fee, traditionally calculated. Therefore, txn fees should be encoded explicitly, as a withdrawal to OP_TRUE (which the block's miner immediately claims).

With all of this in place, the only requirements for inclusion in a block are these:

1. "Be ACKed" -- The "blinded TxID" of this txns must be member of the "approved candidate" set in the D2 of this block.
2. "Return Change to Account" -- TxOut0 must pay {{ see the note above about the payment format being unknown, and replacing CHECKSIG with TRUE }} to the "critical account" (see D1) that corresponds to the CTIP that was selected as a TxIn.
3. "Return *all* Change to Account" -- Sum of inputs must equal the sum of outputs. No traditional tx fee is possible.

And, don't forget that M6 inherits the requirement (common to both M5 and M6) that the CTIP be selected as an input, and that the CTIP then be updated. In this case, we know that the critical index will be zero, so the new CTIP will be ("this TxID" (NOT blinded), 0). The TxID is NOT blinded because blinding is only for accumulating ACKs.

As a result of these requirements, every single withdrawal-attempt will fail, unless an entry has been added to D2 and "ACKed" a sufficient number of times.






{{ BIP Stuff }}


Header
=======

    BIP: ????
    Layer: Consensus (soft fork)
    Title: Hashrate Escrows (Consensus layer)
    Author: Paul Sztorc <truthcoin@gmail.com>
            CryptAxe <cryptaxe@gmail.com>
    Comments-Summary: No comments yet.
    Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-???????
    Status: Draft
    Type: Standards Track
    Created: 2017-08-14
    License: PD


Abstract
==========

{{ Abstract }}


Motivation
============

{{ edit this }}

* Some users desire the ability to do this. [ Most important reason. ]

1. Protect Bitcoin from campaigns to hard fork the network. Such campaigns represent an existential threat to Bitcoin, as well as an avenue for developer corruption. 
2. Protect Bitcoin from competition from altcoins and spinoffs. Safely allow competing implementations (of *sidechains*). 
3. Maintain the monetary property of "recognizability", by ensuring that Bitcoin can always be recognized as money (regardless of which blockchain network is used).
4. Help with review, by making it must easier for reviewers to ignore bad ideas (and be safely proven wrong when these ideas are tried an succeed). No talking necessary.


Backward compatibility
========================

{{ Usual stuff about soft forks. }}

{{ What old nodes do if they encounter new data. }}

BONUS: Sidechains inherit the "soft fork" safe-upgrade-incentive, as soft forks are the only way to guarantee that the WT^s reported by different clients will continue to match.


Deployment
===========

{{ Stuff about versionbits -- use this as reference: https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#deployment }}


Reference Implementation
==========================

See: https://github.com/drivechain-project/bitcoin/tree/mainchainBMM

Also, for interest, see an example sidechain here: https://github.com/drivechain-project/bitcoin/tree/sidechainBMM


References
============

See http://www.drivechain.info/literature/index.html


Thanks
=========

{{ Thank all the good reviewers }}



Copyright
==========

This document is placed in the public domain.


