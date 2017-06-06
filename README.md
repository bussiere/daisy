# Daisy - a private blockchain where blocks are SQLite databases

# What if...

What if there is a blockchain where only certain nodes, in possession of one of accepted private keys, can add data (i.e. new blocks) to a blockchain, whose blocks are (SQLite) databases, and where those existing nodes can accept new ones into their ranks by signing the candidate's keys in a web-of-trust style?

What if public (government) documents were distributed in this way...? What if Wikipedia was...?

# Usage

When started, Daisy will initialise its databases and install the default (and currently the only one) blockchain. It will then connect to a list of peers it maintains and fetch new blocks, if any.

Since this is a private blockchain, not everyone has the ability to create new blocks. I'm thinking of this as a more of a framework for creating new blockchains. If you want to contribute to the default blockchain (i.e. store data, i.e. add new sqlite databases to the blockchain), run the `./daisy mykeys` command, send me the public key hash to sign, and an explanation / introductory letter saying why and what do you want to do with it, and I'll sign your key and accept it into the blockchain as one of the signatories.

Some possible use cases I've thought of for blockchains where everyone can download and verify the data, but only a few parties can publish:

* Distributing academic articles: create a new blockchain for academic institutions (and Arxiv and such) and allow them to push blocks with articles into the blockchain.
* Distributing municipal and governmental records: each institution / agency could be allowed to publish blocks with records and documents into the blockchain.
* Distributing scientific data: only certified research institutions publish data into the blockchain.
* Distributing sports / betting / lottery results
* Wikileaks, of course
* Making a gigantic world-wide database of e.g. product information: manufacturs could add information about their products, keyed on e.g. UPC codes

# Current status

Basic crypto, block and db operations are implemented, the network part is mostly done. Db queries are pending.

This is mostly alpha quality code.

## ToDo

* Implement nicer error handling when replying to messages
* Refactor db* and blockchain* into struct methods
* Implement the "URL" encoding for block transfers: so the data in the JSON messages isn't block data, but an URL to the block data.
* Implement a Bloom filter for tables in SQL queries, to skip querying blocks which don't have the appropriate tables.
* Implement stochastic guarded block importing: if there apparently is a new block in the network: ask a number of peers if they've seen it before importing it.

## Design notes

*Note:* All this is fluid and can be changed as development progresses.

* Everyone can download the blockchain, only special "miners" can create ones, in a sort-of web-of-trust way. Those nodes who are able to create new blocks are called "signatories." They are in posssion of an accepted private key.
* Block payloads are SQLite database files. Except for special metadata tables, their content is not enforced.
* Blockchain metadata is mostly separate from the block payloads, with some obvious exceptions such as the block hash. Metadata critical for blockchain integrity (like the previous block's hash (merkle)is within the block database)
* Consensus rules for accepting new blocks:
    * The validity of the SQLite files
    * The presence of a special tables named `_meta` and `keys`,
    * The validity of the previous block hash in the `_meta` table
    * The previous block hash is signed with a key which is one of the accepted private keys, i.e. signatories, i.e. which is present in the previous blocks' `_keys` table.
    * The `_keys` table contains new signatory keys additions and revocations. Both operations must be signed by a number of currently valid signatories, where the
      number is given as "1 if height < 149 else floor(log(height)*2)"
    * Longest chain wins.
* Flood-based p2p network: every node can request a list of known connections from the other nodes.
* Each message contains the genesis (root) block hash, so technically multiple chains can safely communicate on the same TCP port

## How blocks are created

Blocks are SQLite database files. Every party in posession of an *accepted private key* can create new blocks and sign them. Blocks are accepted (if other criteria are satisfied) only if they are signed by one of the accepted keys.

New blocks can contain operations which add or remove keys from a (global) list of accepted keys, if they contain a sufficient number of signatures from a list of already accepted keys. See the `_keys` table description in the section on Block metadata.

## Block metadata

### The `_keys` table

This table contains key operations for block creators. Keys can be either accepted or revoked. It is invalid for a `_keys` table to contain both acceptance and revocation records for a single key. Both acceptance and revocation operations require a quorum, where Q different keys which are already accepted sign the hash of the key in question. The number Q is calculated as:

```
  Q = 1 if H  < 149 else floor(log(H)*2)
```

Where `H` is the block height of the block containing these records.

For example, if `Q` is 3, to add a key `K` to the list of accepted keys, there must be exactly 3 records in the `_keys` table pertaining to `K`. Each of the records must contain a valid signature by a different, already accepted key. The key `K` can be then used to sign new blocks immediately after the block which contain this records has been accepted.

A table of quorums required for specific block heights is:

```
  1   1
  149 10
  245 11
  404 12
  666 13
  1097 14
  1809 15
  2981 16
  4915 17
  8104 18
  13360 19
  22027 20
  36316 21
  59875 22
  98716 23
  162755 24
```

E.g. for block 100000, 23 signatures are required to accept a new signature.

# Basic crypto

ECDSA P-256 is used for public key crypto operations.

Strings refered in Daisy as "public keys" are SHA256 hashes of public keys and begin with the string "1:" (the 1 is to indicate a key type, should it need to change in the future). They look like "1:9569f0894e3d2b435a4c49c6a97501f4191b9729ff53be5acee2c7bd4be0e439".