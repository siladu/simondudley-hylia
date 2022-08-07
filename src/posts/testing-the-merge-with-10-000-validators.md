---
layout: layouts/post.njk
title: Testing The Merge with 10,000 Validators
date: 2022-08-06T06:18:23.816Z
---
Message: how to setup 10,000 validators on a testnet

1. Context: merge, web3signer, teku, besu
 a. Merge
 b. w3s+teku+besu stack
 c. Why 10K
2. Technical gotchas
 a. Setting up the keys
 b. deposit queue
 c. syncing the clients?
 d. missed validators
3. Monitoring
 a. Logging throttling
 b. Signature graphs
 c. Alerts

https://ethereum.org/en/upgrades/merge/

https://blog.ethereum.org/2022/07/27/goerli-prater-merge-announcement/

https://goerli.launchpad.ethereum.org/en/

https://docs.web3signer.consensys.net/en/latest/


* goerli eth
* teku with initial state from infura
* synching besu, bonsai
* eth2-val-tools keystores --source-mnemonic "${VALIDATORS_MNEMONIC}" --source-min 0 --source-max 3 --insecure --out-loc generated-keys
* ulimit -n 65536
* devnet_deposits.sh 2000 3000

# Technical Gotchas

The easiest way to setup a _single_ validator is by using https://goerli.launchpad.ethereum.org.
The problem with this is that you have to sign each deposit submission in MetaMask. Unless you are a fan of [cookie clicker](http://orteil.dashnet.org/cookieclicker/), you probably don't want to click 10,000 times.

I got pointed to this post which shows how you can script a deposit with [eth2-val-tools](https://github.com/protolambda/eth2-val-tools): 
https://hackmd.io/dFzKxB3ISWO8juUqPpJFfw#Creating-a-validator-deposit

Great, now where do I get Goerli ETH from for 10K validators...that's 10,000 * 32 = 320,000 ETH (worth $500K at the time of writing)...good job it's not real ETH!
I'll leave how I sourced the testnet ETH as an exercise for the reader ;)

