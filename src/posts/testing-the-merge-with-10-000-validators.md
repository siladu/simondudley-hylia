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