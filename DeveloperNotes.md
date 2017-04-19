# Developer Notes
This is just a rough draft, there is much more to be added.


## Language used in Drivechains project:

### `Mainchain`
https://github.com/drivechain-project/bitcoin/tree/mainchain

The mainchain is Bitcoin, with added functionality to support Drivechains. The work that is intended to
be merged upstream all goes into the mainchain branch of the Bitcoin fork in this repository.

### `Sidechain`
https://github.com/drivechain-project/bitcoin/tree/sidechain

The sidechain is a fork of Bitcoin, with added functionality to support the sidechain half of Drivechains.
None of the code in the sidechain repository should ever be merged into upstream bitcoin. Some of the 
heavy lifting and code that we wouldn't necessarily want in Bitcoin core, but is needed to support Drivechains
goes here.

### `SCDB`
`SCDB` is short for Sidechain Database. The SCDB primarily keeps track of the verifications
for each sidechain during the current Tau.

### `State Script`
`State Scripts` are created and interpreted by the SCDB in order to represent the verification
state of active Sidechain WT^(s). State scripts contain hex values like opcodes that tell the SCDB
whether a WT^ is being verified or rejected, and to delimit different Sidechains and their WT^(s).

### `wt`
`wt` is the name given to an individual withdrawal attempt from the sidechain back to the mainchain.

### `WT^`
`WT^` is the name given to a transaction created by the sidechain that joins pending wt's
into a single transaction that can be verified by the mainchain.

### `B-WT^`
`B-WT^` is the blinded version (no inputs) of a WT^ that is voted on. A `WT^` can be verified as a
decendent of a certain `B-WT^` after approval by blinding the transaction again and checking the hash.

### `Tau` &tau; 
`Tau` is used to represent a single round / period of waiting and verification for an individual sidechain.
The Tau of the testing sidechain (SIDECHAIN_TEST) is 300 for example. The Tau is calculated by the 
sidechain.GetTau() function in the following way: `return (sidechain.nWaitPeriod + sidechain.nVerificationPeriod);`

During the Tau of a particular sidechain, up to MAX_SIDECHAIN_WTS (3 for now) may be added to SCDB for verification.
Each sidechain has space for up to MAX_SIDECHAIN_WTS during any single Tau. Note that the Tau for each sidechain may
be different. After the waiting period portion of the Tau, miners may begin verifying the WT^(s) submitted. At the end
of a sidechain Tau, if a WT^ has been verified such that it has sufficient workscore it will be paid out on the mainchain.
At the beginning of the next Tau, the SCDB is blank for that sidechain until a new WT^ is shown to the miners. The SCDB 
is amnesic to previous Tau periods.

## SCDB numbers:
### Maximum number of total sidechains:
Each sidechain is assigned a unique number, represented by an 8 bit unsigned integer.
This limits the maximum number of sidechains to 256.

### State script size:
The OP_RETURN and state script version prefix takes up 3 bytes.
Each WT^ added to a state script takes up around 2 bytes. The max number of WT^s per Sidechain
is 3, so a maximum of 6 bytes will be used per sidechain. Bitcoin will reject a script if it
is larger than 10,000 bytes, meaning up to ~1,666 WT^s on ~555 Sidechains can be represented.

Size of the State script if the three testing sidechains have 3 WT^s each (the maximum):
20 KB

## Known issues and improvements to be made:
Please see https://github.com/drivechain-project/bitcoin/issues
and submit any new issues that you discover.
