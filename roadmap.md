Roadmap
=========



Setup / Example
------------------

A full list of requirements.

Keypairs in this example:

1,"1CAqUCZEjVpWUqWsTKHbd83SYp5FzHKoLQ","L4fwp8Zz6Bya53MQvZd7pjxoLqmUZPQSuCSB5DhxexGT9ASset6L"
2,"1BEqHeMMMQ9pYbUDK2Ym4MqerjR9DoQp9U","L37q5tUDGHHiA8hY9VBEnU2Pj6G2cjh3zzHoDa8mHQvqu58BY9ju"
3,"1Q9YoreXAJVfNpxdKrMt1rsyAAVK85fbg2","L1huLYEDMSxR9JAbvCStfsjuFYcSWfcsCh3f2VMTPpYitX2i9GnU"

* Sidechain Address: 4PzhhUPpXDHifBsAtoh1E9S2wVx3u9ZPG7
* "scriptPubKey": {
    "asm": "057461626c65057461626c65e007e007e803f401 OP_SIDECHAIN",
    "hex": "14057461626c65057461626c65e007e007e803f401c0",
    "type": "nonstandard"
  },


4,"12TvhiV6Hjveh5X243Pb24BiitcDZtaT5U","L36DwrD1f4fcXSf8tAuUsahH21hWqt9BRDH4v4GS4NavsYJofw14"
5,"1GbQLhfCgXr9YXs1Kxz9NSEWae4m1dm8yX","L3SyrXjxrkWc3T6GQkrNv5vsgBgUQpafmsUDECLo5ZLuXzd8nvxh"
6,"14JkbiUhxrQV38kwKKrHWkRwdgBownLtdT","KyTQYzAdmQ8azTWdKFtahtXa32Xpp64vZCRnoeiZXRD8XEEn3QbE"

7,"16Ywiei5D3fR95okt4kC8GkZbdBhXe4vP","L4JqdG5oBVKpN1VMrYwzg78P956ftSsU1HEqk2tQLhHUbijuTJN2"
8,"1PCM7byPBV3Tg3cojpagn5ppaPL3x7WETH","L5j9uxtzsGe9pFpEKztJh5gYpocW6m6DixVi4bwtQBhJihdTbznV"


Roadmap Table
-----------------------

(Note: [m] refers to activity on the mainchain, [s] refers to sidechain.)

| Step #   | Title                              | Description                                                                                                                                                                                                                                                                                                                                                                                                                    | Complete? / Status |
| -------- | ---------------------------------- | -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------                                                                          | ---------          |
| 1        | Keys, Sidechain Adr/Script         | See Above                                                                                                                                                                                                                                                                                                                                                                                                                      | 100%               |
| 2        | Create Sidechain                   | [1] [m] Create a sidechain object (container, btc address)... [2] [m] Miner Database, which keeps track of sidechains and their parameters.                                                                                                                                                                                                                                                                                    | 100%               |
| 3        | Deposit to Sidechain               | [1] [m] Send 0.2 BTC from 1CAq... to 4Pzh..., Send 0.3 BTC from 1BEq... to 4Pzh..., Send 0.4 BTC from 1Q9Y.. to 4Pzh.... [2] These transactions must be discovered by the MinerDB (step 2), and they must also encode a unique address. [3] For example, the first transaction (0.2 BTC) must somehow encode the value "12TvhiV6Hjveh5X243Pb24BiitcDZtaT5U".  (It has been determined that OP_RETURN is safe to use for this.) | 100%               |
| 4        | Receive                            | [1] 10 block waiting period (dampen reorg effects). [2] [s] Watch mainchain for the txns in Step 3. Credit 0.2 BTC to 12Tv..., 0.3 to 1GbQ..., and 0.4 to 14Jk...                                                                                                                                                                                                                                                              | 100%               |
| 5        | Sidechain Game                     | [s] 0.3 BTC transfers from 1GbQ.. to 12Tv.. and this transfer is done in a novel way (impossible with Bitcoin). For example, rock paper scissors betting game.                                                                                                                                                                                                                                                                 | 0% - Low Priority  |
| 6        | Withdrawal Request                 | [1] 12Tv wants to take 0.4 [s] BTC from the sidechain to [m] 16Yw.. [2] 1Q9Y wants to take his [s] 0.4 BTC back as well, to [m] 1PCM..                                                                                                                                                                                                                                                                                         | 100%               |
| 7        | Canonical WT^ Format               | Must be a deterministic formula for constructing a CoinJoin-like transaction, which selects unspent outputs in Step 3 and sends them to the Addresses specified in Step 6.                                                                                                                                                                                                                                                     | 100%               |
| 8        | Sidechain Initiates Withdrawal     | After certain withdrawal and timing conditions are met, sidechain calls mainchain to initiate a withdrawal. Includes [s] commands for "GetCurrentWithdrawalRequest" and "GetSidechainPrivkey", which may be run by [m].                                                                                                                                                                                                        | 100%               |
| 9        | Voting (SPV Proofs, in Coinbase)   | Adding the txID of the CoinJoin-like txn of Step 7. First, this txn is added to a coinbase. Then, miners "upvote" or "downvote" it, committing their work to an interpretation of the withdrawal's validity.                                                                                                                                                                                                                   | 95%                |
| 10       | OP Code                            | OP CheckWorkScore Verify. This is the code which "locks" the outputs of Step 3. The CoinJoin-like txn of Step 7 can be included in a [m] block (now that it has enough "work"). The transaction moves the coins away from OP CWSV outputs, and locks them in "normal" outputs. The transfer, from [s] back to [m], is now complete.                                                                                            | 99%                |
| 11       | Merged Mining                      | ~~Sidechain header must be modified to support merged-mining, as was done with Namecoin.~~ We invented [our own customized form of merged-mining](http://www.truthcoin.info/blog/blind-merged-mining/)! Also, header should occasionally contain the txID of Step 7.                                                                                                                                                                                                                                      | 90%                 |
| 12       | GUI                                | Interface for tracking the process. "Transfers" tab in Sidechain-Qt (and plural "Sidechains" tab in Bitcoin-Qt). Upper view: Incoming and Outgoing sections, listing origin and destination addresses. Lower view: WT^ explorer, listing all WT^ in reverse chronological order, identifying them by tx-id (expanding to reveal all WTs in the WT^).                                                                           | 100%                |


Notes
---------

* If vote "fails", Canonical aggregation should try again. Unclear how this should be handled, programmatically.
* Vote should not trigger at all, in the (rare) event that there are no withdrawal attempts whatsoever.
* Need to limit size of WT^ (at least, to the size 1 MB), and implement a fee sorting-and-ranking algorithm.


Resources
------------

Visit [drivechain.info/literature](http://www.drivechain.info/literature/index.html) for more.

### BIP Documentation

* See https://github.com/drivechain-project/docs#mainchainbmm
* Driect link: https://github.com/drivechain-project/docs/blob/master/drivechain-bips.md

### Blog Posts

1. [Drivechains blog post](http://www.truthcoin.info/blog/drivechain/)
2. [Drivechains OP code](http://www.truthcoin.info/blog/drivechain-op-code/)

### Videos

1. [Sidechain Risks](https://www.youtube.com/watch?v=0goYH2sDw0w&index=1&list=PLw8-6ARlyVciNjgS_NFhAu-qt7HPf_dtg)
2. [Sidechain Privatization](https://www.youtube.com/watch?v=xGu0o8HH10U&list=PLw8-6ARlyVciMH79ZyLOpImsMug3LgNc4&index=1)


