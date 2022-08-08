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

The easiest way to setup a *single* validator is by using https://goerli.launchpad.ethereum.org.
The problem with this is that you have to sign each deposit submission in MetaMask. Unless you are a fan of [cookie clicker](http://orteil.dashnet.org/cookieclicker/), you probably don't want to click 10,000 times.

I got pointed to this post which shows how you can script a deposit with [eth2-val-tools](https://github.com/protolambda/eth2-val-tools): 
https://hackmd.io/dFzKxB3ISWO8juUqPpJFfw#Creating-a-validator-deposit

In addition to the instructions in the post you also need to actually output the validator keys so they can be uploaded to the validator client or in our case Web3Signer. 

The following command should achieve this:
`eth2-val-tools keystores --source-mnemonic "..." --source-min 0 --source-max 10000 --insecure --out-loc generated-keys`

> A note about --insecure: this was a testnet-only convenience we allowed ourselves to reduce the time it takes to decrypt the keys as they are loaded into web3signer upon startup

Attempting to generate 10,000 keys like this, you will quickly run into a "too many open files" error. My default ulimit is 256 file descriptors, so let's up that: 
`ulimit -n 65536`

Ha ok, now there's multiple pages of very painful looking Go stacktraces and my Go-fu is too poor to debug what appears to be an issue with parallel processing.

Let's look at what files are being output:

```shell
$ ls generated-keys-insecure
keys lodestar-secrets nimbus-keys prysm pubkeys.json secrets teku-keys teku-secrets
```

*eth2-val-tools* in its benevolent convenience outputs a variety of client-friendly output formats. Web3Signer is great friends with teku, so it makes sense to reuse the teku format: a list of keystores and associated password files. After a successful hack involving commenting out non-teku related code in eth2-val-tools, I could generate the files in two batches of 5000. However, I decided for a slightly more robust and repeatable solution.

The biggest batch I could generate without hacking the code was 1000. I created a script to generate these smaller batches and stitch them together. I wanted this to work for any number of keys, not specifically 10,000 which wasn't quite as trivial as I first imagined. Here's the crux of what I ended up with:

```shell
# eth2-val-tools cannot handle creating more than 1000 keys worth of files in one run
MAX_SLICE=1000
for ((SLICE_START = SOURCE_MIN;
      SLICE_START < SOURCE_MAX;
      SLICE_START = SLICE_START + MAX_SLICE))
do
  FULL_SLICE_END=$(( $SLICE_START + $MAX_SLICE ))
  SLICE_END=$(( $SOURCE_MAX < $FULL_SLICE_END ? $SOURCE_MAX : $FULL_SLICE_END ))

  echo "Creating keys and secrets for path slice ${SLICE_START} ${SLICE_END}"

  eth2-val-tools keystores \
    --source-mnemonic "${VALIDATORS_MNEMONIC}" \
    --source-min ${SLICE_START} \
    --source-max ${SLICE_END} \
    --insecure \
    --out-loc ${SCRATCH_DIR}/generated-keys-${SLICE_START}

  cp -rf ${SCRATCH_DIR}/generated-keys-${SLICE_START}/teku-keys/ ${SCRATCH_DIR}/keys
  cp -rf ${SCRATCH_DIR}/generated-keys-${SLICE_START}/teku-secrets/ ${SCRATCH_DIR}/secrets
done
```

These files could then be bulk loaded with the appropriate Web3Signer command:
https://docs.web3signer.consensys.net/en/latest/HowTo/Use-Signing-Keys/#keystore-files

Keystores uploaded, deposit script at the ready...great, now where do I get Goerli ETH from for 10K validators...that's 10,000 * 32 = 320,000 ETH (worth $500K at the time of writing)...good job it's not real ETH!
I'll leave how I sourced the testnet ETH as an exercise for the reader ;)

The only thing left to consider was the validator deposit queue. I didn't want to spam the queue with 10,000 validators since validator activation is throttled for security reasons. Doing that could block the queue for a signiciant amount of time, preventing other would-be testers who maybe just wanted to activate a single validator. 

Another factor was how well our test infrastructure would hold up to this many keys. We could do with a steady ramp up which afforded us time to scale up should the need arise. I tentatively started sending batches of 1000 every couple of days. With the 3 second sleep per deposit built into the script, this took about 90 mins per batch.

```shell
deposits.sh 0 1000
deposits.sh 1000 2000
...
```

The pending validator queue was about 4-5000 when I started this and remaining steady. That meant it would take a few days until the batch was fully activated.
After a couple of batches, on a Monday morning I discovered another user had done exactly what I tried to avoid: spammed the queue, it was now 15,000+ pending validators, even bigger than the mainnet queue! It would be weeks before all our validators activated now. This is not a job for the impatient!

Once the 10,000 validators finally activated though, it was all the more sweeter for waiting. Our stack coped well with the load. We didn't need to scale out, although following the final 1000 validators, we did need to scale the teku node up slightly from our original instance type due to CPU occasionally maxing out.

For those familiar with AWS lingo, our final merge-ready setup was *besu* on a **t3.xlarge**, *teku* on a **c6a.2xlarge** and *web3signer* on a **t3.large**. We know from experience with other testnet setups that besu and teku live quite happily together on one instance.

### Postscript

After running the deposit script ten times - post-script if you will - our teku metrics were showing 24 validators with a status of "UNKNOWN". This means that they never made it into the deposit contract. This can be verified by seeing if this RPC returns a result or a 404:

```shell
curl http://localhost:5051/eth/v1/beacon/states/head/validators/<publickey>
```

I put the stray 24 keys down to long-running script times making it hard to spot what was probably a network disconnection. Bash to the rescue again to write a simple script to generate the keys and call curl for each key. 
*eth2-val-tools* has this convenience function for simply printing out the public keys:

```shell
eth2-val-tools pubkeys --source-min 0 --source-max=10000 --validators-mnemonic "..."
```