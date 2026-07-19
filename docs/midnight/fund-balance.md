# `ows fund balance` — Midnight balances (unshielded, shielded, dust)

**Purpose:** read a Midnight wallet's balances from the indexer — the public unshielded
(Night) UTXOs, the private shielded (Zswap) coins, and the DUST fee ledger — for any
`midnight:<network>`.

```sh
ows fund balance --wallet demo --chain midnight:preview
```

A `midnight:*` chain routes to the Midnight indexer instead of the MoonPay path the other
chains use. The unshielded balance needs only the stored address, so it's always shown; the
**shielded** and **DUST** balances need the wallet's key material, from `OWS_PASSPHRASE` (an
owner passphrase or an api-key token). With `OWS_PASSPHRASE` unset the empty passphrase is tried,
as the signing commands do, so a wallet imported with `wallet import` (which sets no passphrase)
needs nothing set. When no credential opens the wallet, the shielded section is left out, the
dust section shows only its NIGHT-registration summary with `DUST seed: unavailable`, and the
command prints:

```
note: set OWS_PASSPHRASE to read Midnight shielded/dust balances
```

An `OWS_PASSPHRASE` that is set but does not open the wallet is an error, not a fallback.

## What it prints

Amounts print to **stdout** as `{amount:>24} {token_type}`; status, addresses, and headers go
to **stderr**, so balances pipe out cleanly. A Preview run (values illustrative):

```
[ows-midnight] syncing balances from indexer (may take a while on first run)…
Addresses:
  Unshielded: mn_addr_preview1dwv2rta0a2skyhrvukaw2q9r2sq6yc4jhj63rf7afxpkrrv6g35q4y8xms
  Shielded:   mn_shield-addr_preview1…
  Dust:       mn_dust_preview1…

Unshielded balances:
              1000000000 0000000000000000000000000000000000000000000000000000000000000000

Shielded balances:
                  500000 0x5cd36fe5ed159246257b476bb67bc8266290581ef3844fcabd7c134b3f8b03ea

Dust status (fees):
  NIGHT UTXOs: total=3 registered=2 unregistered=1
  Fee mode: generationless DUST (can be derived from unregistered NIGHT inputs)
  DUST seed: available
  DUST UTXOs: 4
  DUST balance (specks): 12345678900000000
  DUST balance: 12.345678900000000 (best-effort, wall-clock time)
```

The `Addresses` block re-encodes the stored address to the target network's HRP and adds
shielded/dust when a key is available (else `(unavailable)`). Unshielded token types are the
indexer's `tokenType` as sent (NIGHT is 64 zeros); shielded ones are the coin's token type as
`0x`-prefixed hex. An empty section says so — `none — no unshielded tokens found
for <address> on <chain_id>` / `none — no unspent shielded coins found after full sync`.

## The credential

`OWS_PASSPHRASE` carries one of two things, resolved before syncing:

- **An owner envelope passphrase** — decrypts the packed `MNK1` role seeds and builds the
  `MidnightCryptoProvider` that derives the shielded/dust keys.
- **An api-key token** (prefix `ows_lib::key_store::TOKEN_PREFIX`) — routes through the same
  policy-enforcing channel as `sign-message` / `sign-transaction`, so a scoped token reads
  shielded/dust balances without the raw passphrase.
- **Unset** — the empty passphrase is tried; when it doesn't open the wallet, the shielded and
  DUST balances are left out, with the `note:` above.

The keys are unsealed at the account's stored index (the last segment of its derivation path), so
the shielded and dust keys always belong to the stored unshielded address. The passphrase is
removed from the environment and the unsealed key is dropped as soon as the provider is built, so
neither lives through the sync; the provider itself holds the role seeds and the keys derived
from them until the command ends.

A private-key wallet has no Midnight account (see [wallet.md](./wallet.md)), so the command fails
with `wallet has no account for chain`.

## The three balance streams

### Unshielded (Night)

From the indexer's UTXO set for the address, summed per token type. Needs no key, resumes from
its snapshot but always catches up over the WebSocket (it has no at-tip fast path), and drives the
DUST-registration summary below.

### Shielded (Zswap)

OWS subscribes to the full, unfiltered `zswapLedgerEvents` stream and replays it **locally**
through the crypto provider (`fold_shielded`) into a full spendable wallet state
(`ShieldedWalletState` / `ZswapLocalState`, keyed by nullifier); balances sum its coins. The
viewing key never leaves the process, and the number is **authoritative**: it sums the wallet's
own unspent coins, tracked by nullifier. First sync is slow (full replay); the snapshot cache
(below) makes later runs fast.

### Dust (fees)

The dust-fee section shows only when the **network's dust ledger is live** — decided at run time,
not by network name. OWS probes the indexer's `dustLedgerEvents` tip cursor: a positive `maxId`
means an active ledger (shown); a missing, empty, or unreachable stream hides it. So dust appears
wherever the ledger is active — Preview/Preprod today, **mainnet automatically once its ledger
goes live**, no OWS change. The probe needs no wallet key (the tip is chain-global) and fails safe.

The ledger is replayed through the provider (`fold_dust`) and the balance is computed against the
wall clock as read when the command starts, since DUST decays — hence
`best-effort, wall-clock time`. The
NIGHT-registration summary (`total`/`registered`/`unregistered`) comes from the public UTXOs, so
it shows even without a key, and picks the fee mode: *generationless DUST* when unregistered NIGHT
inputs exist, *DUST spend proofs* otherwise.

## Concurrency

The three streams are independent subscriptions with independent caches, so they sync
concurrently, each on its own thread; wall-clock cost is the **slowest** stream, not the sum, and
a long fold in one stream never leaves another stream's socket unread. A block-height read plus a per-stream live-tip probe gate the shielded/dust fast-paths and
the stale-snapshot check; the unshielded stream has no fast path and always catches up.

## Snapshot cache

Each stream persists its **source state** (UTXO set / Zswap wallet state / dust ledger — never a
derived balance) to disk, so a warm run resumes instead of replaying from genesis:

```
{vault}/chains/midnight/cache/{unshielded|shielded|dust}/{wallet_id}/<hash>.json
```

The `<hash>` fingerprints `(indexer_url, chain_id, key)`, so a snapshot is never reused across a
different indexer, network, or key. Every snapshot also records the **genesis block hash** of the
chain it was synced against, and each stream discards a snapshot whose hash differs from the
indexer's current one (`block(offset: { height: 0 }) { hash }`): a testnet reset keeps the network
id but starts a new chain, and once the new chain's event ids pass the saved cursor the tip check
below no longer sees the reset. A snapshot with no recorded hash is discarded as well; if the
genesis query fails, the snapshot is kept and checked on the next run.

On resume the shielded/dust streams **validate the snapshot against the live stream tip**
(`tip_verify`) by probing the indexer's current `maxId`:

- **discard** it and replay from genesis when the saved cursor sits *past* the live tip — the sign
  of an indexer reset that rewound the event ledger, where resuming would otherwise report the old
  chain's stale balances;
- **skip the WebSocket catch-up** and return the on-disk state when the cursor already sits *at* the
  live tip (when the tip probe gets no answer, an unchanged block height is accepted instead);
- otherwise replay forward from the saved cursor to catch up.

A shielded or dust snapshot the live events do not fit is stale whatever those checks said: when
the next event fails to fold into it (its commitment tree rejects the insertion index), it is
discarded and the stream replays from genesis. Snapshots are written to a temporary file and
renamed into place, so an interrupted run never leaves a truncated one behind.

`OWS_MIDNIGHT_SYNC_CACHE=0` disables disk caching.

## Configuration

The GraphQL indexer URL comes from `~/.ows/config.json` under `rpc["midnight:<network>"]` (the
WebSocket URL is derived from it); defaults ship for mainnet/preview/preprod. Sync timeouts
(HTTP 30s, WS connect 30s, WS idle 90s, stall 120s) are fixed, not environment-configurable.

## Environment variables

| Variable | Effect |
|---|---|
| `OWS_PASSPHRASE` | Owner passphrase **or** api-key token; unlocks shielded/dust (`LWS_PASSPHRASE` is read when it is unset). Unset → the empty passphrase is tried; if that fails, the shielded and DUST balances are left out. Set but wrong → error. |
| `OWS_WALLET` | Default for `--wallet`. |
| `OWS_MIDNIGHT_SYNC_CACHE` | `0`/`false` → disable the disk snapshot cache (always re-sync). |

## The code path

- CLI: `commands/fund.rs::balance` detects a `midnight:*` chain, resolves the credential, builds
  the optional `MidnightCryptoProvider`, and calls `ows_midnight::print_fund_balance`.
- Display + concurrency: `ows-midnight/src/fund_balance.rs`.
- Streams (`wallet_sync/`): `unshielded.rs`, `shielded/` → `shielded/vk_hidden/` (the local
  Zswap replay), `dust/`.
- Cache + gating: `cache_io.rs` (which also reads `OWS_MIDNIGHT_SYNC_CACHE`), `shielded/cache.rs`,
  `shielded/vk_hidden/sync_cache.rs`, `dust/sync_cache.rs`, `tip_verify.rs`; the unshielded
  snapshot code lives in `unshielded.rs`.
- Fixed sync timeouts: `midnight_env.rs`.
