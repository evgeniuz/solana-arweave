# solana-arweave

## Description

The solana-arweave bridge will store Solana ledger (blocks along with containing transactions) in Arweave.

Key considerations:

1. We should be able to query transactions from Arweave by pubkey, signature, blockhash, slot;
2. Querying speed matters more than storage space;
3. As much data as possible should be stored on Arweave chain (i. e. less off-chain data).

### Design

Right now Arweave only allows to query data by block hashes (which are random, hence can't be used for indexing) and by tags using ArQL (https://github.com/ArweaveTeam/arweave-js#arql). Size of tags is a bottleneck here, according to Arweave yellow paper (https://www.arweave.org/yellow-paper.pdf), their size is limited to 2048 bytes., which means if we just store slots (8 bytes) and block hashes (32 bytes) for indexing we can fit about 50 Solana blocks into an Arweave block. If we store more than that, we won't be abble to efficiently query them.

We also need a way to query by pubkey/signatures. This will be achieved by creating special "indexing" blocks on Arweave that will have pubkeys/signatures in their tags and pointers to actual "transaction" blocks in their data. E. g. to query for pubkey we'll get all "index" blocks with that pubkey in tags, get "transaction" blocks from their data, then get actual transactions from those "transaction" blocks.

This approach will require additional blocks with tags only, but will make for faster querying and will store all indexing information on Arweave chain. Another advantage of this approach is that we can optionally skip creating index blocks and instead store this index locally in a key-value store. This may be useful for users who are concerned with Arweave cost. That way they still can store Solana ledger on Arweave, but index to query it by pubkey/signatures will be local.

"Index" blocks will be created once tags are full (to efficiently use available space), that means latest few blocks will be unindexed, but it's easy and fast to just iterate over them. Transaction data will be stored in binary Solana wire serialization, compressed with a gzip or a similar algorithm.

Bridge will monitor Arweave wallet balance and when it's positive will resume operation from last saved block. We can query Arweave for latest block that was saved before stopping.

Working with Solana is straightforward, bridge will get blocks sequentially using JSON RPC API directly in binary wire serialization, which then be preserved as is on Arweave chain.

### Misc

The bridge will be MIT licensed and implemented using Go, this allows to compile it to static binary, that can be easily deployed/distributed on major operating systems.
