---
layout: layouts/post.njk
title: Reducing Storage in Besu
date: 2024-03-26T07:20:38.092Z
tags:
  - dev
  - besu
  - ethereum
  - blockchain
---
We introduced two new storage-saving features in [Besu version 24.3.0](https://github.com/hyperledger/besu/releases/tag/24.3.0): 
1. `--Xbonsai-limit-trie-logs-enabled`
2. `storage x-trie-log prune`

Details of the announcement here:
[https://wiki.hyperledger.org/display/BESU/Limit+Trie+Logs+for+Bonsai](https://wiki.hyperledger.org/display/BESU/Limit+Trie+Logs+for+Bonsai)

A﻿s of [Besu version 24.6.0](https://github.com/hyperledger/besu/releases/tag/24.6.0) `--bonsai-limit-trie-logs-enabled` has been made production ready and enabled by default (unless you're using `sync-mode=FULL` to maintain an archive node).

## Bonsai and trie logs

Besu's popular BONSAI data storage format enables a node to only store the world state for the latest, block greatly reducing storage requirements. It achieves this by maintaining state diffs between blocks called trie logs. More details about BONSAI and trie logs: [https://consensys.io/blog/bonsai-tries-guide](https://consensys.io/blog/bonsai-tries-guide)

When a block is imported into Besu, we also create a trie log (state diff) and store it in TRIE_LOG_STORAGE. Besu uses RocksDB for its database and stores trie logs in a column family aptly called TRIE_LOG_STORAGE:


```bash
|--------------------------------|-----------------|-------------|-----------------|------------------|
| Column Family                  | Keys            | Total Size  | SST Files Size  | Blob Files Size  |
|--------------------------------|-----------------|-------------|-----------------|------------------|
| BLOCKCHAIN                     | 1468107011      | 716 GiB     | 104 GiB         | 612 GiB          |
| VARIABLES                      | 6062            | 102 KiB     | 102 KiB         | 0 B              |
| ACCOUNT_INFO_STATE             | 243501676       | 11 GiB      | 11 GiB          | 0 B              |
| ACCOUNT_STORAGE_STORAGE        | 1121004167      | 50 GiB      | 50 GiB          | 0 B              |
| CODE_STORAGE                   | 37852944        | 12 GiB      | 12 GiB          | 0 B              |
| TRIE_BRANCH_STORAGE            | 1906989010      | 140 GiB     | 140 GiB         | 0 B              |
| TRIE_LOG_STORAGE               | 514             | 99 MiB      | 71 KiB          | 99 MiB           |
|--------------------------------|-----------------|-------------|-----------------|------------------|
| ESTIMATED TOTAL                | 4777461384      | 931 GiB     | 319 GiB         | 612 GiB          |
|--------------------------------|-----------------|-------------|-----------------|------------------|
```


The trie logs are retained to cope with chain reorgs and RPC queries that may need to roll back the state to an older block. 

In Proof of Stake on Ethereum mainnet, after each block is finalized, trie logs older than the finalized block are no longer required* so it is safe to remove them both from the node's and the network's point of view: a finalized block cannot be reorged.

It is therefore redundant to store trie logs older than the finalized block. As it turns out over time trie logs can take up a reasonable amount of space, getting into the hundreds of GBs if a node has been running for many months.

We were surprised to discover that the large number of transactions on mainnet accounts for about **3GB of TRIE_LOG_STORAGE disk growth per week** if left unchecked.

\*This is true but in reality, we store the most recent 512 blocks by default. This number is somewhat arbitrary but enables us to easily support use cases other than Ethereum mainnet without impacting mainnet disk size greatly (it's only ~100MB). In fact, in a Proof of Stake chain, if a long non-finality period were to occur we may have >512 unfinalized blocks in which case the blocks wouldn't be pruned until they were finalized.


## Limiting trie log growth

The main problem to solve is limiting the continuous growth of the trie logs. We need to retain at least the most recent trie logs since the last finalized block. 

### Pruning queue

This is achieved by [adding the trie logs to a pruning queue](https://github.com/hyperledger/besu/blob/e56a37da110e78a843460f6d6580521b787401cb/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/bonsai/trielog/TrieLogPruner.java#L183) whenever they are [stored in the database](https://github.com/hyperledger/besu/blob/e56a37da110e78a843460f6d6580521b787401cb/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/bonsai/trielog/TrieLogManager.java#L78-L79).

It was preferable to avoid using the database for this queue, both for performance and complexity reasons. Instead, [we maintain an in-memory queue of references to the recent trie logs](https://github.com/hyperledger/besu/blob/e56a37da110e78a843460f6d6580521b787401cb/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/bonsai/trielog/TrieLogPruner.java#L50-L51).

As a separate operation, [the pruner is triggered](https://github.com/hyperledger/besu/blob/e56a37da110e78a843460f6d6580521b787401cb/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/bonsai/trielog/TrieLogPruner.java#L184) which prunes all eligible trie logs: [anything in the queue that is older than the finalized block](https://github.com/hyperledger/besu/blob/e56a37da110e78a843460f6d6580521b787401cb/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/bonsai/trielog/TrieLogPruner.java#L142-L145). This is a cheap operation since deleting in RocksDB involves marking it with a tombstone to be deleted later during regular compaction. Nevertheless, we perform the "add to queue" and "trigger prune" operations [asynchronously off the main thread](https://github.com/hyperledger/besu/blob/e56a37da110e78a843460f6d6580521b787401cb/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/bonsai/trielog/TrieLogPruner.java#L181) since we don't want to impact the response time of the Engine API.

### Forks and orphaned trie logs

Trie logs have a one-to-one mapping with blocks, but not _canonical_ blocks. Trie logs are created for blocks that may later become forks due to reorgs. In Besu, they are also created every time our node proposes a block, which happens multiple times during the four-second block proposal window as we try to build the best block given a dynamic transaction pool. [In the pruning code](https://github.com/hyperledger/besu/blob/2eca4d5a4e0535655195ea1da58e34a7570e176b/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/bonsai/trielog/TrieLogPruner.java#L89), this is referred to as an "orphaned" trie log.

The in-memory queue is [a Multimap with block number as the key and a list of block hashes as the value](https://github.com/hyperledger/besu/blob/2eca4d5a4e0535655195ea1da58e34a7570e176b/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/bonsai/trielog/TrieLogPruner.java#L50-L51). This list represents all the forks and the orphaned trie logs for a given block number, only one of which is the canonical block. When we prune blocks beyond a certain block height we're also pruning canonical block's trie logs, forks and orphaned trie logs in the same operation. Orphaned blocks are treated slightly differently during the queue preloading discussed next.

### Preloading and pruning the gap
One side-effect of the in-memory queue approach is that whenever you restart Besu, you lose your queue state. This creates a gap in the pruning since we only add to the prune queue when new trie logs are stored. This is [mitigated by _preloading_ the queue at startup](https://github.com/hyperledger/besu/blob/e56a37da110e78a843460f6d6580521b787401cb/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/bonsai/trielog/TrieLogPruner.java#L69-L101). We also take the opportunity to [perform a single prune operation](https://github.com/hyperledger/besu/blob/e56a37da110e78a843460f6d6580521b787401cb/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/bonsai/trielog/TrieLogPruner.java#L96), so typically, the gap that was created is preloaded and immediately pruned. The default values for how many trie logs to preload and prune are [tuned to cover this gap](https://github.com/hyperledger/besu/blob/e56a37da110e78a843460f6d6580521b787401cb/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/worldstate/DataStorageConfiguration.java#L62-L63).

When we start Besu, the queue preloading operation sources data from TRIE_LOG_STORAGE so [only has access to the block hashes](https://github.com/hyperledger/besu/blob/2eca4d5a4e0535655195ea1da58e34a7570e176b/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/bonsai/trielog/TrieLogPruner.java#L78). We need to [query the blockchain object to determine the block number](https://github.com/hyperledger/besu/blob/2eca4d5a4e0535655195ea1da58e34a7570e176b/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/bonsai/trielog/TrieLogPruner.java#L84-L86). Canonical blocks and forks will appear in the blockchain object, however orphaned trie logs will not. [In this case, we simply immediately prune the orphaned trie logs ](https://github.com/hyperledger/besu/blob/2eca4d5a4e0535655195ea1da58e34a7570e176b/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/bonsai/trielog/TrieLogPruner.java#L88-L92)rather than adding them to the queue; a freebie for the pruner!

Once this limit trie logs feature is enabled, _future_ growth of the trie logs is limited which saves at least **3GB per week** of trie log growth on disk for mainnet. Then the question becomes how do we deal with the _backlog_ of trie logs for existing nodes?

## Pruning the backlog

We could have had the pruner running in a background thread chipping away at the backlog but that is an extra moving part of complexity and potentially makes overall node performance more unpredictable.
Instead, we opted to [offer an offline command](https://github.com/hyperledger/besu/blob/e56a37da110e78a843460f6d6580521b787401cb/besu/src/main/java/org/hyperledger/besu/cli/subcommands/storage/TrieLogSubCommand.java#L114-L120) as a one-off operation which - provided the `Xbonsai-limit-trie-logs-enabled` feature remains enabled - users should only need to run once to clear their backlog.

The TRIE_LOG_STORAGE column family stores the block hash as the key. This makes it difficult to look up by block _number_ which is what we want to do to prune below a certain block number. 

A naive approach is to load _all_ the trie log keys into memory and look up their hashes in the blockchain to determine the block numbers. [Tests showed that this took 3 hours to prune a 3.5-month-old node](https://github.com/hyperledger/besu/pull/6188#issuecomment-1851081569). We knew some nodes would be significantly older than this. A [nice optimisation suggested by Gary Schulte](https://github.com/hyperledger/besu/pull/6188#discussion_r1424650025) was to stash the recent trie logs we wanted to retain, clear the whole TRIE_LOG_STORAGE column family and repopulate with just the recent trie logs. This enabled us to bring the prune command down to a few seconds no matter how many we were deleting, which was a huge win - thanks, Gary!

## User results

We were keen to get feedback from users and see how much space they saved. To support this, we spent a few extra seconds measuring the TRIE_LOG_STORAGE column family size before and after during execution of the prune command and printed it out to users:
> Prune ran successfully. We estimate you freed up 263 GiB! 🚀

[Sharing was encouraged on social channels](https://www.reddit.com/r/ethfinance/comments/1b9gurw/comment/ktvthnu) and users happily obliged which was [awesome feedback to receive](https://old.reddit.com/r/ethstaker/comments/1bcb6rl/new_besu_update_is_awesome/). We were pleasantly surprised by the numbers. The Besu team only had access to nodes that were around three or four months old with ~70GB of trie logs. Some users with long-running nodes were able to save as much as **263GB** in seconds by running the prune command!

In combination with `--Xbonsai-limit-trie-logs-enabled` saving ~3GB per week, a minimally configured (sync-mode=CHECKPOINT) Besu mainnet node at the time of writing (March 2024) takes up about **930GB on disk with 7-8GB per week growth**.