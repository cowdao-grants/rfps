# RFP-02: MEV Blocker bundle merging tracker

## Preamble

## Background

[MEV Blocker](https://cow.fi/mev-blocker), which is part of CoW DAO's product offering, is the leading private RPC endpoint in terms of adoption to no small degree thanks to it's superior searcher coverage and MEV refund performance compared to other RPCs.
While malicious MEV attacks such as front-running or sandwiching are avoided altogether, MEV opportunities that don't directly harm the user, such as backruns, are auctioned off to a permissionless set of searchers, while 90% of those proceeds are refunded to the user (or a specified `refundRecipient`).

As part of [CiP-40](https://snapshot.org/#/cow.eth/proposal/0x67aaf4f3e14f833ea496a5cf29d3c58cd12e36a72b7f36c60393083b87422070), MEV Blocker started to prescribe formal rules which builders have to adhere to when they connect to receive MEV Blocker transaction flow.
Violation of these rules leads to slashing.

The rules are specified in https://docs.cow.fi/mevblocker/builders/rules and amongst others meant to ensure that user refunds are maximised.

In particular, the rules enforce that - in the case of multiple searcher bids - as many MEV Blocker backruns as possible get included.
That is, MEV Blocker backruns which pay a refund should get prioritised over competing external backruns which don't pay the user but make the overall block more valuable (e.g. backruns directly submitted to the builder or identified by the builder internally).

## Goal

Provide an open source implementation that (greedily) verifies for each landed transaction user refunds were maximised and alert violations.

### Specification

- For each MEV Blocker transaction that was included in a block:

  - Identify the position `p` of transaction in block
  - For each bundle submitted for this transaction in that block up until 11s into the block:

    - Simulate bundle (with target transaction in position `p`)
    - If simulation passes, identify payment to builder `$`
    - Otherwise discard bundle

  - Sort bundles descending by `$` identified in step above
  - Store state `s` of current block after position `p` (after target transaction was applied)
  - For each of the sorted bundles

    - Simulate only the backrun part in position `p+1`
    - If simulation passes, update `s` to the state after the simulation, increment `p` and add backrun tx hash to list of expected txs
    - Otherwise ignore the backrun and continue

  - Check that each expected backrun tx hash was included in the target block (and was followed by a refund transaction to the refund recipient)
  - In the case a backrun wasn't included, re-simulate it at the end of the block
    - If it passes continue (the builder is just bad)
    - If it fails alert (likely some other transaction used the backrun and didn't optimally pay back the user)

This logic should ensure that at least a greedy bundle merging algorithm was applied and that any backrun that would have still been possible after the user transaction wasn't "used" by some other transaction in the block.

The latter would suggest that the builder included another transaction instead which relied on the target transaction but didn't pay a refund to the user.
If such a transaction simply pay 20% of its value to the validator it will make the block more valuable than the original MEV Blocker transactions (which pays 10% to validators).
However, such behavior is specifically disallowed by the rules.

### Ressources

All MEV Blocker transactions and bundles are posted with a slight delay in the this Dune table: `mevblocker.raw_bundles` (cf. [sample query](https://dune.com/queries/3114264))

The code that syncs MEV Blocker data with dune is also open source: https://github.com/cowprotocol/mevblocker-dune/tree/main/src

Simulation of multiple transactions (e.g. all transaction of a block leading up to p + the bundle) can be done using non standard [`trace_callMany`](https://www.quicknode.com/docs/ethereum/trace_callMany) RPC calls or more advanced tools such as (Temper)[https://github.com/EnsoFinance/temper] or [Tenderly](https://docs.tenderly.co/forks/using-forks-with-ethers-js)

For more background information on MEV Blocker, cf. https://docs.cow.fi/mevblocker

### Deliverables

Either:

- a script that can be run periodically (e.g. as a cron job) to check and alert violations for a specific time-frame

Or

- a realtime system that monitors for new blocks, and processes them as the required Dune data becomes available

Additionally:

- Development container and/or other relevant automation (e.g. Dockerfile, .env, etc.) to aid running in production
- Comprehensive documentation on the logic of the script and how to run it
