# DriveChain RPC commands

countsidechaindeposits

createbmmcriticaldatatx

createcriticaldatatx

getbmmproof

listpreviousblockhashes

listsidechainctip

listsidechaindeposits

receivewtprime

receivewtprimeupdate




## countsidechaindeposits

Returns the number of deposits (for nSidechain) currently cached. Note that this doesn't count all sidechain deposits, just the number currently cached by the node.

Arguments:
1. "nsidechain"      (numeric, required) The sidechain number

Examples:
> drivenet-cli countsidechaindeposits "nsidechain"
> curl --user myusername --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "countsidechaindeposits", "params": ["nsidechain"] }' -H 'content-type: text/plain;' http://127.0.0.1:8332/




## createbmmcriticaldatatx

Create a BMM request critical data transaction

Arguments:
1. "amount"         (numeric or string, required) The amount in BTC to be spent.
2. "height"         (numeric, required) The block height this transaction must be included in.
Note: If 0 is passed in for height, current block height will be used3. "criticalhash"   (string, required) h* you want added to a coinbase
4. "nsidechain"     (numeric, required) Sidechain requesting BMM
5. "ndag"           (numeric, required) DAG number

Examples:
> drivenet-cli createbmmcriticaldatatx "amount", "height", "criticalhash", "nsidechain", "ndag"
> curl --user myusername --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "createbmmcriticaldatatx", "params": ["amount", "height", "criticalhash", "nsidechain", "ndag"] }' -H 'content-type: text/plain;' http://127.0.0.1:8332/




## createcriticaldatatx

Create a critical data transaction

Arguments:
1. "amount"         (numeric or string, required) The amount in BTC to be spent.
2. "height"         (numeric, required) The block height this transaction must be included in.
3. "criticalhash"   (string, required) h* you want added to a coinbase

Examples:
> drivenet-cli createcriticaldatatx "amount", "height", "criticalhash"
> curl --user myusername --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "createcriticaldatatx", "params": ["amount", "height", "criticalhash"] }' -H 'content-type: text/plain;' http://127.0.0.1:8332/




## getbmmproof

Get the BMM proof (txoutproof) of an h* BMM commit transaction on the mainchain. Used by the sidechain (optionally) to double check BMM commits before connecting a sidechain block

Arguments:
1. "blockhash"      (string, required) mainchain blockhash with h*
2. "criticalhash"   (string, required) h* to create proof of

Examples:
> drivenet-cli getbmmproof "blockhash", "criticalhash"
> curl --user myusername --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "getbmmproof", "params": ["blockhash", "criticalhash"] }' -H 'content-type: text/plain;' http://127.0.0.1:8332/




## listpreviousblockhashes

List the 5 most recent mainchain block hashes. Used by sidechains to help search for BMM commitments.

Arguments:

Examples:
> drivenet-cli listpreviousblockhashes 
> curl --user myusername --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "listpreviousblockhashes", "params": [] }' -H 'content-type: text/plain;' http://127.0.0.1:8332/




## listsidechainctip

Returns the crtitical transaction index pair for nSidechain

Arguments:
1. "nsidechain"      (numeric, required) The sidechain number

Examples:
> drivenet-cli listsidechainctip "nsidechain"
> curl --user myusername --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "listsidechainctip", "params": ["nsidechain"] }' -H 'content-type: text/plain;' http://127.0.0.1:8332/




## listsidechaindeposits

List the most recent cached deposits (for nSidechain). Optionally limited to count. Note that this does not return all sidechain deposits, just the most recent deposits in the cache.

Arguments:
1. "nsidechain"      (numeric, required) The sidechain number
2. "count"           (numeric, optional) The number of most recent deposits to list

Examples:
> drivenet-cli listsidechaindeposits "nsidechain", "count"
> curl --user myusername --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "listsidechaindeposits", "params": ["nsidechain", "count"] }' -H 'content-type: text/plain;' http://127.0.0.1:8332/




## receivewtprime

Called by sidechain to announce new WT^ for verification

Arguments:
1. "nsidechain"      (int, required) The sidechain number
2. "rawtx"           (string, required) The raw transaction hex

Examples:
> drivenet-cli receivewtprime 
> curl --user myusername --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "receivewtprime", "params": [] }' -H 'content-type: text/plain;' http://127.0.0.1:8332/




## receivewtprimeupdate

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
> drivenet-cli receivewtprimeupdate 
> curl --user myusername --data-binary '{"jsonrpc": "1.0", "id":"curltest", "method": "receivewtprimeupdate", "params": [] }' -H 'content-type: text/plain;' http://127.0.0.1:8332/
