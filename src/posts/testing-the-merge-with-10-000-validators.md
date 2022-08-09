---
layout: layouts/post.njk
title: Testing The Merge with 10,000 Validators
metaTitle: Testing The Merge with 10,000 Validators
metaDesc: Testing Web3Signer and The Merge with 10,000 Validators
date: 2022-08-09T03:20:35.332Z
tags:
  - dev
  - ethereum
  - blockchain
  - merge
---
On the verge of [the Goerli merge](https://blog.ethereum.org/2022/07/27/goerli-prater-merge-announcement/), the final Ethereum testnet to be transitioned to proof-of-stake, I wanted to share how I've been involved in testing our products as a blockchain protocol engineer at ConsenSys.

If you have no idea what The Merge or even Ethereum is, then [start here](https://ethereum.org/en/upgrades/merge/) (and welcome to the rabbit hole!)

One of the products my team is responsible for is [Web3Signer](https://github.com/ConsenSys/web3signer), an enterprise-ready key management and signing service specialising in the Ethereum proof-of-stake beacon chain. Web3Signer is an addition to an Ethereum consensus layer client, for example, [Teku](https://github.com/ConsenSys/teku). We need a consensus client to meaningfully test Web3Signer. In the context of The Merge and a post-merge system, we also need an execution layer client such as [Besu](https://github.com/hyperledger/besu).

Besu, Teku and Web3Signer, all being ConsenSys products, are a natural fit for our test stack. Since Web3Signer is designed to support institutional stakers, we chose 10,000 keys as a large but realistically sized deployment.

This post will discuss some of the ~~fun that was had~~ technical issues encountered while commissioning such a setup.

## 10,000 keys

The easiest way to set up a *single* validator is by using https://goerli.launchpad.ethereum.org.
The problem with this is that you have to sign each deposit submission in MetaMask. Unless you're a fan of [Cookie Clicker](http://orteil.dashnet.org/cookieclicker/), you probably don't want to click 10,000 times.

I got pointed to this post which shows how you can script a deposit with [eth2-val-tools](https://github.com/protolambda/eth2-val-tools): 
[https://hackmd.io/dFzKxB3ISWO8juUqPpJFfw#Creating-a-validator-deposit](https://hackmd.io/dFzKxB3ISWO8juUqPpJFfw#Creating-a-validator-deposit)

In addition to the instructions in the post you also need to output the validator keys that are derived from your mnemonic so they can be uploaded to the validator client, or in our case Web3Signer. 

The following command should achieve this:

```bash
eth2-val-tools keystores --source-mnemonic "..." \
  --source-min 0 --source-max 10000 \
  --insecure --out-loc generated-keys
```

> A note about --insecure: this was a testnet-only convenience we allowed ourselves to reduce the time it takes to decrypt the keys as they are loaded into Web3Signer upon startup

Attempting to generate 10,000 keys like this, you will quickly run into a "too many open files" error. My default ulimit is 256 file descriptors, so let's up that: 
```bash
ulimit -n 65536
```

Ha ok, now there's multiple pages of very painful-looking Go stacktraces and my Go-fu is too poor to debug what appears to be an issue with parallel processing.

Let's look at what files are being output:

```bash
$ ls generated-keys-insecure
keys lodestar-secrets nimbus-keys prysm 
pubkeys.json secrets teku-keys teku-secrets
```

*eth2-val-tools* in its benevolent convenience outputs a variety of client-friendly output formats. Web3Signer is great friends with Teku, so it makes sense to reuse the Teku format: a list of keystores and associated password files. After a successful hack involving commenting out code unrelated to the Teku output in eth2-val-tools, I could generate the files in two batches of 5000. However, I decided on a slightly more robust and repeatable solution.

The biggest batch I could generate without hacking the code was 1000. I created a script to generate these smaller batches and stitch them together. I wanted this to work for any number of keys, not specifically 10,000 which wasn't quite as trivial as I first imagined. Here's the crux of what I ended up with:

```bash
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
[https://docs.web3signer.consensys.net/en/latest/HowTo/Use-Signing-Keys/#keystore-files](https://docs.web3signer.consensys.net/en/latest/HowTo/Use-Signing-Keys/#keystore-files)

## The Queue

Keystores uploaded, deposit script at the ready...great, now where do I get Goerli ETH from for 10K validators...that's 10,000 * 32 = 320,000 ETH (worth $500K at the time of writing)...good job it's not real ETH!
I'll leave how I sourced the testnet ETH as an exercise for the reader ;)

The only thing left to consider was the validator deposit queue. I didn't want to spam the queue with 10,000 validators since validator activation is throttled for security reasons. Doing that could block the queue for a significant amount of time, preventing other would-be testers who maybe just wanted to activate a single validator. 

Another factor was how well our test infrastructure would hold up to this many keys. We wanted a steady ramp up which afforded us time to scale up should the need arise. I tentatively started sending batches of 1000 every couple of days. With the three-second sleep per deposit built into the script, this took about 90 minutes per batch.

```bash
./deposits.sh 0 1000
./deposits.sh 1000 2000
...
```

When I started sending batches the pending validator queue was about 5000 and remained steady. That meant it would take a few days until the batch was fully activated.
After a couple of batches, on a Monday morning I discovered another user had done exactly what I tried to avoid: spammed the queue, it was now 15,000+ pending validators, even bigger than the mainnet queue! It would be weeks before all our validators activated now. This is not a job for the impatient!

Once the 10,000 validators finally activated though, it was all the sweeter for waiting. Our stack coped well with the load. We didn't need to scale out, although following the final 1000 validators, we did need to scale the Teku node up slightly from our original instance type due to CPU occasionally maxing out.

For those familiar with AWS lingo, our final merge-ready setup was *Besu* on a **t3.xlarge**, *Teku* on a **c6a.2xlarge** and *Web3Signer* on a **t3.large**. We know from experience with other testnet setups that Besu and Teku live quite happily together on one instance.

### Postscript

After running the deposit script ten times - post-script if you will - our Teku metrics were showing 24 validators with a status of "UNKNOWN". This means that they never made it into the deposit contract. This can be verified by seeing if this RPC returns a result or a 404:

```bash
curl http://localhost:5051/eth/v1/beacon/states/head/validators/<publickey>
```

I put the stray 24 keys down to long-running script times making it hard to spot what was probably a network disconnection. Bash to the rescue again to write a simple script to generate the keys and call curl for each key. 
*eth2-val-tools* has this convenience function for simply printing out the public keys:

```bash
eth2-val-tools pubkeys --source-min 0 --source-max=10000 \
  --validators-mnemonic "..."
```