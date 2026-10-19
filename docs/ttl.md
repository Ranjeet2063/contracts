# TTL Values and Storage Expiry

## What is TTL in Soroban?

Soroban persistent storage entries have a **Time-To-Live (TTL)** measured in **ledgers**. Once an entry's TTL reaches zero, the Stellar network archives it and it becomes inaccessible until restored via a fee-paying restore operation.

## Current TTL Configuration

Persistent storage entries in `alert-registry` are categorized across ten distinct keys, detailed in [docs/storage.md](storage.md):

1. `DataKey::Alert(id)` — individual alert configs
2. `DataKey::RemovedAlert(id)` — tombstones for removed alerts to enable incremental sync (see #360)
3. `DataKey::AlertActive(id)` — per-alert active flag for fast lookups without full config deserialization
4. `DataKey::OwnerIndex(address)` — list of alert IDs owned by a given address
5. `DataKey::OwnerLiveCount(address)` — running count of non-removed alerts owned by address
6. `DataKey::ContractIndex(address)` — list of alert IDs watching a given target contract
7. `DataKey::ContractLiveCount(address)` — running count of non-removed alerts targeting a contract
8. `DataKey::DeactivationCursor(address)` — cursor tracking pagination position during batch deactivation
9. `DataKey::PendingTransfer(id)` — pending ownership transfer proposals awaiting acceptance
10. `DataKey::AdminSuspended(id)` — administrative suspension marker set by `deactivate_alert_by_admin`

### Default TTL

Most persistent entries use the standard **default TTL of 17,280 ledgers (~24 hours)**:

```rust
// DEFAULT_TTL = 17_280  (~24 hours at 5 s/ledger)
env.storage().persistent().extend_ttl(&key, DEFAULT_TTL, DEFAULT_TTL);
```

### Non-Default TTL Paths

Not all persistent keys follow the standard `DEFAULT_TTL` lifecycle:
- **`PendingTransfer(id)`**: Uses `ALERT_TRANSFER_EXPIRY_LEDGERS` (17,280 ledgers, ~24 hours). The entry is written when `propose_alert_transfer` is called and explicitly expires after `ALERT_TRANSFER_EXPIRY_LEDGERS` ledgers if not accepted, rejected, or cancelled.
- **Tombstones (`RemovedAlert(id)`)**: Written with `DEFAULT_TTL` when an alert is removed. Unlike active alert records, tombstones are never renewed or bumped by `bump_alert`, intentionally expiring after ~24 hours (see #360).
- **`DeactivationCursor(address)`**: Has its own dedicated TTL path managed exclusively by `deactivate_all_alerts`. When a batch deactivation cannot finish all alerts within a single transaction, the cursor is saved with `DEFAULT_TTL`. If the deactivation is completed in a subsequent batch or if an alert is removed, the cursor is deleted immediately.
- **`ContractLiveCount(address)`**: Maintained incrementally alongside `ContractIndex(target)`. It is lazily initialized and extended with `DEFAULT_TTL` during contract index mutations, and also refreshed as part of `touch_alert` when present.

The `watcher-registry` uses **instance storage**, which has its own TTL managed
by the network. No registry entrypoint extends it as a side effect, so see
[Keeping the watcher registry alive](#keeping-the-watcher-registry-alive) below.

## Keeping the Watcher Registry Alive

All `WatcherRegistry` state (admins, watchers, timelock) lives in a single
instance entry. Its TTL is only refreshed when a transaction touches the
contract — and a registry whose watchers are registered once and then rarely
changed may go untouched long enough for the entry to be archived, taking the
whole registry offline until it is restored for a fee.

`bump_instance_ttl` is the keep-alive entrypoint:

```rust
// Anyone may call this — no auth, no state change.
watcher_client.bump_instance_ttl();
```

It extends the instance TTL to `INSTANCE_BUMP_AMOUNT` (535 680 ledgers, ~31
days) whenever it has dropped below `INSTANCE_BUMP_THRESHOLD` (17 280 ledgers,
~24 hours), and is the `WatcherRegistry` analogue of `AlertRegistry::bump_alert`.

**Operators should schedule a keeper to call it well inside the ~31-day
window** — a call every few days is cheap and leaves ample margin. Any other
transaction against the contract in the meantime does *not* refresh the TTL on
its own, so do not rely on incidental traffic.

## Keeping the Alert Registry Instance Alive

`AlertRegistry` also keeps state in its instance entry: the admin, the alert ID
counter (`NEXT_ID`), the per-owner, per-contract and global limits, the pause
flag and the watcher registry address. If that entry is archived, every alert
becomes unreachable until it is restored, even though the alerts themselves
are persistent entries.

Unlike `WatcherRegistry`, AlertRegistry extends its instance entry on **every
write to instance storage**: each `register_alert` / `batch_register_alert`
(through the ID counter) and every admin setter (`initialize`,
`transfer_admin`, `pause`, `unpause`, `set_per_owner_alert_limit`,
`set_per_contract_alert_limit`, `set_global_alert_limit`,
`set_watcher_registry`, `clear_watcher_registry`). Each extends the instance
TTL to `INSTANCE_BUMP_AMOUNT` (535 680 ledgers, ~31 days) once it has dropped
below `INSTANCE_BUMP_THRESHOLD` (17 280 ledgers, ~24 hours), the same values as
`WatcherRegistry`.

For quiet periods with no registrations or admin changes, it exposes the same
permissionless keep-alive:

```rust
// Anyone may call this — no auth, no state change.
alert_client.bump_instance_ttl();
```

**Operators should have a keeper call it every few days**, alongside
`WatcherRegistry::bump_instance_ttl`. Reads and alert updates do not touch the
instance entry's TTL.

## Configurable TTL via `bump_alert`

Callers can extend the TTL of any alert up to the **protocol maximum of
535 680 ledgers (~31 days)** by calling `bump_alert`:

```rust
// Extend alert 42 to the maximum lifetime (~31 days)
alert_client.bump_alert(&42u64, &MAX_TTL);
```

Request at most `MAX_TTL`. The current build caps larger values at `MAX_TTL`,
but whether they should be capped or rejected is still open (#28), so callers
should not rely on the capping.

The function:
1. Caps the requested TTL at `MAX_TTL` (535 680 ledgers).
2. Extends `Alert`, `AlertActive`, `OwnerIndex`, and `ContractIndex` entries.
3. Emits an `("alert", "bump")` event with `(id, effective_ttl)`.

No auth is required — any address (e.g. an off-chain keeper service) may
bump an alert's TTL without modifying its content.

## Wall-Clock Time Estimate

Stellar ledgers close approximately every **5 seconds**.

| Ledgers | Approximate wall-clock time |
|---------|-----------------------------|
| 100     | ~8 minutes 20 seconds       |
| 1,000   | ~1 hour 23 minutes          |
| 17,280  | ~24 hours (DEFAULT_TTL)     |
| 120,960 | ~7 days                     |
| 535,680 | ~31 days (MAX_TTL)          |

**The default TTL of 17 280 ledgers keeps an alert alive for ~24 hours after
the last write.**  Use `bump_alert` to extend beyond that without modifying
the alert's content.

## When Does the TTL Reset?

Every mutating call refreshes the **whole set** of entries an alert depends on
through a single internal helper, `touch_alert`:
- `DataKey::Alert(id)`
- `DataKey::AlertActive(id)`
- `DataKey::OwnerIndex(owner)`
- `DataKey::OwnerLiveCount(owner)` (extended when present; see [docs/storage.md](storage.md))
- `DataKey::ContractIndex(target)`
- `DataKey::ContractLiveCount(target)` (extended when present)
- `DataKey::AdminSuspended(id)` (extended when present)

Mutators that rewrite the alert call `persist_alert`, which writes the config
and its `AlertActive` flag and then calls `touch_alert` with `DEFAULT_TTL`, so
no mutator can extend one entry and forget another.

| Function | Entries extended |
|---|---|
| `register_alert` | Full set, to `DEFAULT_TTL`; initializes `OwnerLiveCount` and `ContractLiveCount` |
| `update_alert` | Full set, to `DEFAULT_TTL` |
| `update_webhook` | Full set, to `DEFAULT_TTL` |
| `update_label` | Full set, to `DEFAULT_TTL` |
| `propose_webhook` | Full set, to `DEFAULT_TTL` |
| `confirm_webhook` | Full set, to `DEFAULT_TTL` |
| `cancel_webhook_proposal` | Full set, to `DEFAULT_TTL` |
| `deactivate_alert_by_admin` | Full set, to `DEFAULT_TTL`; writes and extends `AdminSuspended(id)` |
| `unlock_alert_by_admin` | Full set, to `DEFAULT_TTL`; removes `AdminSuspended(id)` |
| `accept_alert_transfer` | Full set with the **new** owner's `OwnerIndex`/`OwnerLiveCount`, to `DEFAULT_TTL`; the old owner's index and counter are rewritten and extended; `PendingTransfer(id)` is removed |
| `reject_alert_transfer` | Deletes `PendingTransfer(id)`; alert's own entries are untouched |
| `cancel_alert_transfer` | Deletes `PendingTransfer(id)`; alert's own entries are untouched |
| `propose_alert_transfer` | `PendingTransfer(id)` only, to `ALERT_TRANSFER_EXPIRY_LEDGERS`; alert's own entries are untouched |
| `update_target_contract` | Full set with the **new** target's `ContractIndex`/`ContractLiveCount`, to `DEFAULT_TTL`; the old target's index and counter are rewritten and extended |
| `deactivate_all_alerts` | Full set for each deactivated alert, to `DEFAULT_TTL`; updates and extends `DeactivationCursor(caller)` if batch is incomplete, or deletes it if complete |
| `renew_alert_ttl` | Full set, to `DEFAULT_TTL` — data unchanged |
| `bump_alert` | Full set, to the requested TTL (capped at `MAX_TTL`) — data unchanged |
| `remove_alert` | `Alert(id)` and `AlertActive(id)` deleted; `RemovedAlert(id)` tombstone created and extended to `DEFAULT_TTL`; `OwnerIndex(owner)`, `OwnerLiveCount(owner)`, `ContractIndex(target)`, `ContractLiveCount(target)` updated and extended to `DEFAULT_TTL`; `PendingTransfer(id)` and `DeactivationCursor(owner)` removed |
| `remove_alert_by_admin` | Same as `remove_alert` |

Read-only functions (`get_alert`, `get_alerts_for_contract`, `get_alerts_by_owner`, `get_non_removed_alert_count`, etc.) do **not** extend the TTL. For full details on storage tiers and data structures, see [docs/storage.md](storage.md).

## Implications

- **Inactive alerts expire eventually.** An alert that is registered but never updated will be archived after `DEFAULT_TTL` (17,280 ledgers, ~24 hours).
- **Watchers must keep alerts alive.** Any off-chain service relying on alert data should periodically call `update_alert` (or a dedicated bump function) to prevent expiry.
- **Archived entries are not deleted.** They can be restored via `RestoreFootprintOp`, but this requires paying a fee and is not currently automated.

## Recommendations

For production use, the TTL should be increased significantly. Common choices:

| Use case | Suggested TTL (ledgers) | Wall-clock |
|----------|------------------------|------------|
| Short-lived alerts | 17,280 | ~24 hours |
| Standard alerts | 120,960 | ~7 days |
| Long-lived alerts | 535,680 | ~31 days |

To change the TTL, update `DEFAULT_TTL` in `contracts/alert-registry/src/lib.rs`.
Every mutator refreshes alert entries through `persist_alert` / `touch_alert`,
so the constant is the single place to change:

```rust
// Example: extend to ~7 days
pub const DEFAULT_TTL: u32 = 120_960;
```

## Further Reading

- [Soroban State Archival](https://developers.stellar.org/docs/learn/encyclopedia/storage/state-archival)
- [Soroban Storage TTL Reference](https://developers.stellar.org/docs/build/smart-contracts/getting-started/storing-data)
