---
SWIP: 74
title: Coordinated contract version transition
author: Andrew Macpherson (@awmacpherson)
status: Draft
type: Process
created: 2026-10-21
---

# Coordinated contract version transition

## Abstract

We propose `Redistribution` contract redeployment and activation be coordinated around a *cutover timestamp*, announced significantly in advance and communicated with scriptable migration instructions that operators can put in a cronjob. We also suggest several tiers of client improvements that make this transition easier and lower risk for operators, with the best case being that operators only need to upgrade before the cutover timestamp to ensure a smooth transition with minimal earnings downtime. 

We also discuss how to handle the case where `StakeRegistry` is redeployed as well as `Redistribution`. The proposal does not cover how to redeploy `Postage` or upgrade the wire protocol.

## Motivation

The dependency and immutability structure of Swarm's smart contracts is such that any redeployment of any of the contracts requires a new `Redistribution` contract that points to it. When the new `Redistribution` contract is deployed and activated, revenue is redirected, and nodes must connect to the new contract in order to continue earning. If there is a delay between when revenue is redirected and when nodes transition to the new contract (in either order), those nodes cannot earn during the transition period. In the past, earnings downtime of up to six days has been observed due to contract transitions.

Disruptions to operator incentives make it more costly to be an operator and could affect service quality, especially during the transition period. Such disruptions are to be avoided where possible. Therefore, we should aim for the gap between revenue redirection and client transition to be as short as possible.

## Specification

When Swarm is to undergo a protocol upgrade that necessitates the deployment of a new `Redistribution` contract (and optionally, a new `StakeRegistry` contract), the admin coordinating the upgrade announces a timestamp, the *cutover timestamp*, at which the chain protocol transition is to take place.

The announcement MUST take place significantly in advance of the timestamp being announced, and may coincide with the release of new client versions. The interval between the announcement time and the timestamp being annouced is called the *lead time.* The lead time MUST be long enough for operators to make appropriate preparations (see [Client](#Client)). The lead time MAY depend on the urgency of the change. As a starting point, we suggest:

* 30 days (routine upgrades)
* 7 days (security-critical upgrades)

The admin commits to deploy the new contracts sufficiently far in advance of the cutover timestamp for deployment validation and integration of the new addresses into client default configurations. The new contracts MUST land before the release of new client versions, which in turn MUST occur with enough time for operators to upgrade before the cutover timestamp. If, for whatever reason, there seems to be a significant risk that this will fail, the cutover SHOULD be rescheduled.

The admin commits to broadcasting a *cutover transaction* at the cutover timestamp. The cutover transaction is a single transaction that executes the following steps:

1. Pause the old `Redistribution` contract.
2. Revoke the `REDISTRIBUTOR_ROLE` of the old `Redistribution` contract on the other three contracts.
3. Grant `REDISTRIBUTOR_ROLE` of the new `Redistribution` contract on the other three contracts.

If the `StakeRegistry` is also being redeployed, it executes the following steps:

1. Pause the old `Redistribution` and `StakeRegistry` contracts.
2. Revoke the `REDISTRIBUTOR_ROLE` of the old `Redistribution` contract on `Postage` and `PriceOracle` contracts.
3. Grant `REDISTRIBUTOR_ROLE` of the new `Redistribution` contract on `Postage`, `PriceOracle`, and the new `StakeRegistry` contract.

The cutover transaction MUST be broadcast with adequate gas bid to ensure a low risk of a long delay between transmission and execution.

Client behaviour depends on the extent to which the protocol transition is integrated into the software:

* *Single-version.* Client implements just one version of the chain protocol. Scheduling must be executed by the operator, who closes the old client version and starts up the new client version at the cutover timestamp.
* *Dual-version.* Client implements both the old and new versions of the protocol. The client continues operating on the old protocol version until the cutover timestamp, at which point it switches out its chain protocol client for the new version. The operator does not need to provide external scheduling infrastructure.

In summary, the following steps are completed *in order*:

1. Deploy new `Redistribution` contract and validate deployment.
2. Integrate new deployment ABI to client configurations.
3. Release new client and announce cutover timestamp. Urge operators to upgrade clients.
4. (At cutover timestamp.) Broadcast cutover transaction.

## Rationale

*Why a timestamp?* To minimise the length of the transition period for a contract upgrade, nodes need a globally observable signal, known to all clients in advance, on which to schedule their transition from the old to the new contracts. The simplest way to define a globally observable, predictable scheduling event is to specify a time and date.

*Why a cutover transaction?* To avoid downtime or earnings fragmentation, revenue must be redirected — the old `Redistribution` contract paused and the new one wired in — at the same time. To reduce the risk of revenue redirection partially failing, or having different parts land at different times, the two operations should be executed atomically, i.e. in a single transaction.

*Why announce the broadcast time for cutover?* To minimise the earnings gap for individual operators, the signal used to schedule node transition should be as close as possible to the cutover signal. It is impossible to guarantee any particular transaction lands onchain at a specific time, or even at a specific block height. However, in practice the wait time for a properly configured transaction to land onchain is small enough to be economically irrelevant. The simplest way to advertise when the cutover will happen is to commit to broadcast the transaction at a specific time, which may as well be the cutover timestamp. 

This step can be further derisked using onchain timestamp checks; see [Extensions](#Extensions).

## Risks

*Earnings gap.* Clients executing the transition exactly at the advertised timestamp cannot earn until the cutover transaction actually lands. Attempts to interact with the new `Redistribution` contract before the cutover result in transaction reverts, hence missed revenue and wasted gas expenditure. 

Under normal conditions, transaction wait times on Gnosis are typically much less than a minute, making this delay economically irrelevant. The main risk here is therefore from admin misconfigurations or network outages. Operators wishing to derisk this process may optionally poll the old `Redistribution` contract for paused status before shutting down their nodes.

*Unfair freezing from mid-play node shutdown.* A node that executes a shutdown while participating in a round of redistribution, after landing `commit()` but without landing `reveal()`, will have their stake frozen for failure to reveal. If `StakeRegistry` is not being redeployed, that freeze persists into the new version. As mitigation for this, clients currently participating in a round of redistribution should not shut down until their participation in that round has reached a terminal state. For convenience, we recommend that this mitigation be programmed into clients.

### With `StakeRegistry` redeployment

*Stake migration congestion.* If all nodes try to migrate stake as soon as the stake registry is paused, thousands of transactions will flood the mempool, leading to blockspace congestion, fee spikes, and unpredictable delays. To mitigate this, migrating nodes should add a short, randomised delay to the cutover timestamp before attempting a stake migration. Under current network conditions, a maximum delay of a few hours should be sufficient.

Operators who do not wish to voluntarily delay migration face an economic tradeoff in the form of increased gas fees or unpredictable wait times.

*Error-prone stake migration instructions.* With status quo clients, migrating stake requires executing a careful shutdown of the old client before transitioning to the new client. If the operator winds down the old client incorrectly, they cannot easily recover using the new client. We recommend that a stake migration helper be added to clients to reduce the scope for operator error.

## Implementation

### Admin

If the admin broadcasts pause and role granting calls from a Safe wallet, we suggest the use of the Safe [MultiSend](https://github.com/safe-global/safe-core-sdk/blob/main/guides/integrating-the-safe-core-sdk.md#4-create-a-transaction) and [Propose](https://github.com/safe-global/safe-core-sdk/blob/main/guides/integrating-the-safe-core-sdk.md#5-propose-the-transaction-to-the-service) API to programmatically construct and propose the cutover transaction for signing in the Safe UI.

### Instructions for operators

The current bee implementation holds a reference to just one set of smart contracts, hence cannot be used to schedule the client cutover. With a status quo client, operators must fetch the new version of the software and schedule a cron job at the announced timestamp that executes the following steps:

1. Shut down the running client.
2. Start the new version of the software using the same data directory as before.

To mitigate the risk of earnings gap or mid-play shutdown freezes, the following waits can be added to migration scripts:

* *Earnings gap.* Poll `Redistribution` state and wait to observe paused status before shutting down.
* *Mid-play shutdown.* Poll the `/redistributionstate` endpoint, executing the shutdown as soon as `phase` changes from `"reveal"` to `"claim"`.

#### Redeploying `StakeRegistry`

If `StakeRegistry` is being redeployed, operators must follow an additional procedure before shutting down:

1. Poll the state of `StakeRegistry` until paused status is observed. 

   If the recommendation of this proposal to execute the cutover atomically is respected, this always occurs at the same time as pausing `Redistribution`, so the latter can also be used as a signal.

2. Wait an exponentially distributed random time, for instance with the command

   ```bash
   sleep $(python3 -c "import random; print(random.expovariate(0.001))")
   ```

   Alternatively, skip to (3) immediately but be prepared to pay over the odds for gas or be subject to unpredictable delays.

3. Call `DELETE /stake` on the bee API.

#### Recovering from failed stake migration

In the current implementation of bee, if the operator fails to wind down the old client correctly, leaving stake stranded on the old stake registry, the funds cannot be recovered using the new bee version running with vanilla configuration. The team SHOULD provide operators with instructions to recover from this state; for example, by running the new client with an explicit `--staking-address` flag pointing to the old contract. Instructions should be provided for for discovering the address of the old contract.

### Client improvements

To smooth the migration process, we suggest the following features be added to clients:

1. *Clean shutdown.* To mitigate the risk of random unfair freezing due to mid-round shutdown, clients should be given a "clean" shutdown sequence that waits for any currently active play to reach a terminal state before closing.

   This feature would be useful for general node maintenance at any time, not only during protocol transitions.

   If ongoing rounds are allowed to complete on paused `Redistribution` (see [Extensions](#Extensions)), the same logic can be used to perform a clean transition when play overlaps the cutover timestamp.

2. *Pause-gated shutdown.* Poll `Redistribution` for paused status before shutting down. This feature is purely a risk management measure for dealing with gaps between the cutover timestamp and onchain activation. It is not needed with onchain enforcement of cutover (see [Extensions](#Extensions)).

3. *Scheduled ABI transition.* Build the ABI of both versions of the contracts into the client, and load the new ABI and chain protocol into memory and swap out the old one at the cutover timestamp. The old ABI can be removed from the client at the next patch after the cutover.

   Clients need to track some ephemeral state if currently participating in a round of redistribution, and the schema of this state could change after transition to the new contract. The easiest way to handle this state is to wait until that round is completed or until transactions on the old `Redistribution` contract revert with `"Pausable: paused"` before executing the transition.

   In bee, the transition can be implemented by rebuilding and swapping out the `storageincentives.agent` object attached to the node object. The transition primarly affects code in the `storageincentives` package and `storer/sample.go`. We suggest factoring the latter into its own package, perhaps as a subpackage of `storageincentives`, to make this transition cleaner.

   For currently planned upgrades, the redistribution API (2 redistribution endpoints and 5 staking endpoints) can be conserved with some changes to semantics that lock in at the time of transition.

#### With `StakeRegistry` redeployment

1. *Stake migration helper.* Add an endpoint on upgraded clients to migrate stake from an old to a new `StakeRegistry`. A reference to the old `StakeRegistry` must be kept in memory after the cutover in order to execute the migration; it can be discarded after migration.
2. *Stake migration shutdown or transition with randomised delay.* Add a special option or command for shutting down or transitioning ABI at the cutover timestamp and migrating stake after a short, randomised delay. After the cutover timestamp, wait an exponentially distributed random amount of time before executing the migration helper.

## Extensions

* *Allow ongoing round completion in paused `Redistribution`.* To prevent pause interrupting an active round, allow `reveal` and `claim` calls on a paused `Redistribution` contract to proceed unmodified. Funds in postage already reserved for the pot should remain claimable by the final active round (this is the status quo pause behaviour). This allows upgraded clients to cleanly complete an ongoing round and claim earnings before transitioning.

  Since at most a single round could be interrupted by pause, the impact of this change would be extremely minor.

* *Onchain enforcement of cutover scheduling.* If the cutover timestamp could be enforced onchain, the risk of delays between client transition and onchain cutover would be eliminated. The following mechanism was suggested by @mfw78:

  * Give the new `Redistribution` contract admin privileges over the old contracts.
  * If `commit()` is called before the new contract is activated, check the current block timestamp against the (hardcoded) cutover timestamp. If the cutover timestamp has passed, perform the cutover in this transaction with the new `Redistribution` contract as authority.

  This approach adds a single inequality check, and hence a small gas cost, to every call to `commit` (and a more substantial cost to the first call after the transition timestamp has passed). The mild additional complexity in the contract is offset by potential reduction in client complexity, since clients now do not have to deal with the risk of reverts during the transition.