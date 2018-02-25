# Developer Notes
This is just a rough draft, there is much more to be added.

To see the rest of the documentation for the Drivechain project:
https://github.com/drivechain-project/docs

## Language used in Drivechains project:

### `Mainchain`
https://github.com/drivechain-project/bitcoin/tree/mainchainBMM

The mainchain is Bitcoin, with added functionality to support Drivechains. The work that is intended to
be merged upstream all goes into the mainchain branch of the Bitcoin fork in this repository.

### `Sidechain`
https://github.com/drivechain-project/bitcoin/tree/sidechainBMM

The sidechain is a fork of Bitcoin, with added functionality to support the sidechain half of Drivechains.
This code doesn't ever need to be merged into Bitcoin upstream.

### `BMM`
'BMM' stands for blind merged mining which is how Drivechains use the PoW of the Bitcoin mainchain. Refer
to http://www.truthcoin.info/blog/blind-merged-mining/ for more information.

### `SCDB`
`SCDB` is short for Sidechain Database. The SCDB keeps track of the work score
of WT^(s) for each sidechain during the current verification period as well as BMM linking data.

### `wt`
`wt` is the name given to an individual withdrawal attempt from the sidechain back to the mainchain.

### `WT^`
`WT^` is the name given to a transaction created by the sidechain that joins pending wt's
into a single transaction that can be verified by the mainchain.

### `B-WT^`
`B-WT^` is the blinded version (no inputs) of a WT^ that is voted on. Once a B-WT^ has 
sufficient workscore the WT^ with inputs will be created. The WT^ can be verified by 
blinding it again.

## RPC 
### createcriticaldatatx
```
createcriticaldatatx
Create a critical data transaction

Arguments:
1. "amount"         (numeric or string, required) The amount in BTC to be spent.
2. "height"         (numeric, required) The block height this transaction must be included in.
3. "criticalhash"   (string, required) h* you want added to a coinbase

Examples:
> bitcoin-cli createcriticaldatatx "amount", "height", "criticalhash"
> curl --user myusername --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "createcriticaldatatx", "params": ["amount", "height", "criticalhash"] }' -H 'content-type: text/plain;' http://127.0.0.1:8332/
 (code -1)
 ```
 
### listsidechaindeposits
```
listsidechaindeposits
Called by sidechain, return list of deposits

Arguments:
1. "nsidechain"      (numeric, required) The sidechain number

Examples:
> bitcoin-cli listsidechaindeposits "nsidechain"
> curl --user myusername --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "listsidechaindeposits", "params": ["nsidechain"] }' -H 'content-type: text/plain;' http://127.0.0.1:8332/
 (code -1)
```
### receivewtprime
```
receivewtprime
Called by sidechain to announce new WT^ for verification

Arguments:
1. "nsidechain"      (int, required) The sidechain number
2. "rawtx"           (string, required) The raw transaction hex

Examples:
> bitcoin-cli receivewtprime 
> curl --user myusername --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "receivewtprime", "params": [] }' -H 'content-type: text/plain;' http://127.0.0.1:8332/
 (code -1)
```

### receivewtprimeupdate
```
receivewtprimeupdate
Receive an update for a WT^

Arguments:
1. "height"                      (numeric, required) the block height
2. "updates"                     (array, required) A json array of json objects
     [
       {
         "sidechainnumber":n,    (numeric, required) The sidechain number
         "hashWTPrime":id,       (string,  required) The WT^ hash
         "workscore":n           (numeric, required) The updated workscore
       } 
       ,...
     ]

Examples:
> bitcoin-cli receivewtprimeupdate 
> curl --user myusername --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "receivewtprimeupdate", "params": [] }' -H 'content-type: text/plain;' http://127.0.0.1:8332/
 (code -1)
 
 receivewtprimeupdate 10 '[{"sidechainnumber": '1', "hashWTPrime": "'3366e8825f6c0d6da7c870c1ab6b035e1ef1610a6d1e7d7afb972a169c4c648a'", "workscore": '1'}]'
 
```

## More information:

### Maximum number of valid sidechains:
Each sidechain is assigned a unique number, represented by an 8 bit unsigned integer.
This limits the maximum number of sidechains to 256.

### Known issues and improvements to be made:
Please see https://github.com/drivechain-project/bitcoin/issues
and submit any new issues that you discover.

### Useful links:
http://www.drivechain.info/literature/index.html

https://github.com/drivechain-project/

https://bitcoin-hivemind.herokuapp.com/ (sidechains discussion slack channel)

Follow [Paul Sztorc](https://twitter.com/Truthcoin) on Twitter.

Join the IRC channel -- #drivechain-dev on freenode.
