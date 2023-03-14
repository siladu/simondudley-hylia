---
layout: layouts/post.njk
title: The Road to Shanghai (via Austria)
date: 2023-03-14T10:47:49.488Z
tags:
  - dev
---
[Once again](https://www.simondudley.com/posts/testing-the-merge-with-10-000-validators/), we're on the verge of a Goerli hardfork for [Ethereum's next upgrade, Shanghai](https://blog.ethereum.org/2023/03/08/goerli-shapella-announcement). This time around, I was leading the charge from the Besu side and want to share some thoughts about how it's gone so far, some lessons learned and some interesting tools.

The primary feature in the Shanghai upgrade is validator withdrawals, as specified by [EIP-4895](https://eips.ethereum.org/EIPS/eip-4895). Validators are users who stake ETH to secure [Ethereum's Proof of Stake](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/) consensus mechanism and get rewarded in the process. The previous upgrade, The Merge (aka Paris) transitioned the Ethereum protocol from Proof of Work to Proof of Stake and this upgrade completes the job by allowing stakers to withdraw their rewards as well as their full stake.

With Proof of Stake, the Ethereum blockchain is a combination of two systems: 

1. The Consensus Layer (CL) secures and drives the protocol by implementing the Beacon Chain and includes clients such as Lodestar, Nimbus, Prysm, Lighthouse and Teku.
2. The Execution Layer (EL) implements the rest: EVM, transaction processing, data storage/availability and includes clients such as Erigon, Nethermind, Geth and Besu.

The specification for withdrawals on the EL side was simpler than on the CL, and since I work on Besu I will focus on the EL perspective.

For more details about the withdrawals upgrade see: https://consensys.net/shanghai-capella-upgrade/

My overall feeling is this has been a pretty smooth fork which I believe is primarily due to:

1. The EL changes we minimal, especially compared to The Merge.
2. We have benefited greatly from the QA tools and best practices developed over the years that were used in anger during testing The Merge.

I'm sure there are other factors such as having similar teams of developers and testers delivering this as for The Merge.

I transitioned to work full-time on Besu mainnet in the second half of last year and picked up the Withdrawals work in November 2022. At this point we had some "spike" code, a nearly complete prototype implementation. My job was to implement EIP-4895 which meant productionising this spike. This also included an adjacent specification for the Engine API (how the CL drives the EL) https://github.com/ethereum/execution-apis/blob/main/src/engine/shanghai.md

Since The Merge, Besu devs had been hard at work improving stability and performance which meant we started EIP-4895 in earnest later than some other client teams. These client teams were understandably keen to test the new code and so shortly after I picked up the EIP, suddenly we were at the point of being asked to join a Withdrawals devnet.
This created an interesting pressure that I'd not experienced before. Rather than an internal deadline set by a manager, product owner or ourselves, it was a peer-based deadline. 
I use deadline loosely because there was no date but rather a "we can't start a devnet until most clients are ready" kind of vibe. You now have up to eight other client teams waiting for you, and your lack of readiness is all in public of course :)

I had to assess the state of our prototype and decide whether we could join the devnet. Like a lot of software, it followed the 80/20 rule: it was probably 80% complete but that final 20% ended up taking 80% of the time. Unlike most other software products, we can't ship at 80% feature complete because that will result in a misbehaving client; it has to follow the specification precisely. Unlike most other software products, we have a precise specification which is really nice to work from.

At the time I thought it was closer to 90% so decided to deploy the prototype with some fixes onto the first devnet. This put us on a path to keep tweaking a long-lived feature branch as bugs were discovered, rather than concentrating on getting the code onto our main branch. In hindsight, it would have been prudent to get some production-grade pieces onto the main branch first and iterate from there. This would have meant missing out on the early devnets though. Given the quality of the test suites to come, this probably wouldn't have mattered.

Then came the real spanner in the works, the hidden feature with unintended consequences. Since this was the first hardfork to be synchronised across both CL and EL, and CL forks work based on the wall clock time, it was specified that the EL fork should be triggered from the block timestamp instead of the usual block number.
This didn't seem to cause too much trouble for other EL client teams, but it was a real pain to implement in Besu. Part of the reason is that Besu is more than just an Ethereum mainnet client, it also supports other use cases such as Ethereum Classic and private networks, including various Proof of Authority consensus mechanisms. These other use cases rely on and have code intertwined with the concept of forks being triggered by block numbers. I think this was a more difficult change to implement in Besu than EIP-4895 itself. There's still tech debt and clean-up associated with it ongoing now.

By the time we felt pretty stable on the devnets, we now had the job of getting this unwieldy branch merged into main. https://github.com/hyperledger/besu/pull/4818
The branch was a couple of months stale by this point, including some fairly large conflicting refactors. 
Rather than go through a big bang git merge, we decided to piecemeal merge chunks of it onto main, completing the spike's missing test coverage as we went. This code was mostly hidden behind the "Shanghai fork" which acts as a feature toggle (only triggering at a certain timestamp). Out of this, we learned a good pattern for adding code onto main behind a feature toggle. The following order seems to work quite well and could be applied to other types of software:

1. Merge the feature toggle
2. Merge new data types and changes to existing types (in a backwards-compatible way)
3. Merge production-grade (read including unit tests) features one by one hidden behind the toggle as necessary

The process of productionisation took longer than anticipated, as these things are want to do.
There were also quite a few last-minute changes to the Shanghai requirement list, which were small enough to be hard to argue against but add up to quite a time sink.
This meant it was quite a large task getting the productionisation completed and ready for the Edelweiss Interop workshop in Austria. https://blog.ethereum.org/2023/02/07/edelweiss-interop-recap

Edelweiss was a great chance for me to meet the other core devs for the first time, including some members of the besu team.

Besu was in pretty good shape by this point so we were mostly filling gaps in tests. A nice achievement for in-person cross-team working was a member of Lighthouse corralling everyone to agree to implement a new API endpoint, engine_exchangeCapabilities as part of Shanghai: https://github.com/ethereum/execution-apis/pull/364
Within 24 hours, we had this in the Besu codebase: https://github.com/hyperledger/besu/pull/4997

Apart from having great specifications to work from, the other first-class developer experience is the Hive tests, built and maintained by the Ethereum Foundation. A large suite of docker-based end-to-end tests that cover all manner of edge cases and even spin up mini-networks with the various client pairs. These links may eventually go stale but it's so impressive I want to share them:
https://github.com/ethereum/hive
https://hivetests2.ethdevops.io
https://hivewithdrawals.ethdevops.io

The only drawback is that they take a long time to run so you can't effectively wire them into your CI. They run a couple of times a day though and we're looking at how we can integrate alerts with them so we know when we've broken something.
It's quite a big investment in keeping on top of making them all pass, but the value of this regression suite is hard to put a number on: they have found many a bug.\
\
I﻿n all, I've had a wild six months getting to grips with Ethereum mainnet development\
and the interesting processes and tools that come with it. It's even been topped off with my first news interview: https://cointelegraph.com/news/next-stop-shanghai-ethereum-s-latest-milestone-approaches