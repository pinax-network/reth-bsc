# Changelog

All notable changes to the Pinax-maintained Firehose fork of bnb-chain/reth-bsc are documented
here. Entries up to `v0.1.1-fh` were written by StreamingFast in `streamingfast/reth-bsc`, from
which this fork was seeded.

This changelog covers Firehose-specific changes only. For upstream changes, see the
[bnb-chain/reth-bsc repository](https://github.com/bnb-chain/reth-bsc).

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## Unreleased

### Fixed

- Honor `FIREHOSE_DISABLED` when installing the Firehose execution extension, as
  well as when initializing the tracer. Previously a plain archive RPC startup
  could briefly serve requests and then exit because the extension accessed the
  uninitialized tracer. The normal Firehose-enabled path is unchanged.

## dev-049d306-fh3.1-3

### Fixed

- The deterministic Firehose finality introduced in `dev-049d306-fh3.1-2` never engaged: the
  tracer starts before consensus validates the block's header, which is when the block's own
  Parlia snapshot is created, so the resolver found no snapshot and every emitted block fell back
  to LIB `block - 200` (first canary: 33/33 one-blocks). The resolver now reads the **parent's**
  snapshot (`snapshot(parent_hash).vote_data.source`, provided through the new `parent_hash`
  argument in `pinax-network/bnb-reth` tag `bnb-bf76323c-fh3.1-3`). Still identical on every
  node and on the block's own ancestry; the advertised LIB is one block staler (head-3 in steady
  state). Debug logs under target `bsc::firehose` when it returns `None`.

## dev-049d306-fh3.1-2

### Fixed

- Firehose finality is now derived per block from the block's own Parlia attestation
  (`snapshot(hash).vote_data`, the same rule as `parlia_getFinalizedNumber`) instead of this
  node's canonical finalized head, via the new `finalized_for_block` hook in
  `pinax-network/bnb-reth` tag `bnb-bf76323c-fh3.1-2`. The canonical head is node state (vote
  arrival timing) and is not necessarily an ancestor of a side-chain block being traced; on BSC
  fast finality this made readers stamp different LIBs on identical blocks (the merger then filed
  ~50k duplicate one-blocks as "forked"), let LIB step backwards within one reader, and stamped
  fork-branch block 120653743 (`e3d67571`) with the canonical finalized number 120653740, so a
  downstream forkdb marked the fork's 120653740 (`2c97928d`) final (2026-09-08 08:24 UTC).
  Blocks without an attestation are emitted with no finalized ref (fireeth falls back to
  `block - 200`), never with node state.

## dev-049d306-fh3.1-1

Firehose build of upstream `develop` commit `049d3065` ("fix: refine new-payload/fcu flow to
avoid memory accumulation", bnb-chain/reth-bsc #501; upstream `v0.1.2` + 15 commits). First
release cut from an upstream commit rather than an upstream tag, using the
`release/<upstream>-fh` branch model: the StreamingFast Firehose series as of `v0.1.1-fh3.1`
replayed onto `049d3065`.

### Changed

- `reth-*` crates come from `pinax-network/bnb-reth` tag `bnb-bf76323c-fh3.1-1` (branch
  `firehose/develop`): bnb-chain/reth `bf76323c` (what `049d3065`'s lockfile resolves
  `branch = "develop"` to) plus StreamingFast's 17 Firehose hook commits from `bnb-v0.1.1-fh3.1`
  and a regenerated lockfile. Public repository, no token needed.
- Two glue conflicts resolved against upstream changes since `v0.1.1`: the Pasteur contract
  upgrade path now both reports the code change to the state hook (upstream's bsc-qanet root
  fix) and emits the Firehose `on_code_change` event; internal consensus `eth_call`s use
  upstream's `view_call_tx_env()` helper with Firehose tracing suspended around them.
- `dev-*` tags are published as prereleases and never move the `latest` image tag.
- The `v0.1.1-fh3.1-1` clippy fix is not needed here: upstream already rewrote that code.

## v0.1.1-fh3.1-1

First release built from `pinax-network/reth-bsc`. `Cargo.toml` and `Cargo.lock` are identical
to StreamingFast's `v0.1.1-fh3.1` (`56c35604c`), so `--locked` resolves exactly the dependency
bytes StreamingFast shipped: `streamingfast/reth` tag `bnb-v0.1.1-fh3.1` and `streamingfast/evm`
`sf/v0.34.0`. One source file differs (see Fixed).

### Fixed

- `read_all_system_contracts` initialises `dir` at its declaration (clippy 1.98
  `needless_late_init`, which `ci.yml` runs with `-D warnings` on unpinned stable). Same behaviour;
  the identical failure shows on `streamingfast/reth-bsc` `release/0.1.x`.

### Changed

- Release pipeline runs in this repository: images publish to `ghcr.io/<owner>/reth-bsc`
  and the `reth-bsc_linux_amd64` asset attaches to the GitHub release here.
- Docker build and CI can authenticate git for dependencies pinned to the private
  `pinax-network` org (secret `PAT_INTERNAL_REPOSITORIES`, applied to `pinax-network/*` URLs
  only). Byte-identical mirrors of the two StreamingFast dependencies exist at
  `pinax-network/reth` (`release/bnb-0.x`, tag `bnb-v0.1.1-fh3.1`) and `pinax-network/evm`
  (`sf/v0.34.0`) for the day the pins move off `streamingfast/*`.

## v0.1.1-fh

Release ready for prime time


## v0.1.1-fh-beta-13

### Fixed

- Include the SELFDESTRUCT refund when resolving an account's post-transaction balance
  (`streamingfast/bnb-reth` port of `streamingfast/reth` `v2.3.0-fh-7`). On the truly-destroyed
  path (EIP-6780: contract created in the same transaction, or pre-Cancun) revm credits the
  beneficiary in place and records the move only inside its `AccountDestroyed` journal entry — no
  `BalanceTransfer` is pushed — so the journal walk backing the `RewardTransactionFee` and
  `GasRefund` events missed it. A coinbase or sender that received a suicide refund then reported
  an `old_balance` contradicting the `SuicideRefund` event emitted moments earlier.

## v0.1.1-fh-beta-12

### Fixed

- Restored two beta-4/6 fixes dropped during the v0.1.1 rebase: the EIP-2935
  history-storage system call is captured again (`transact_system_call` routes through the
  inspector when tracing), and the canonical block size again excludes blob sidecars.
  (beta-10/11 traces were missing `system_calls` entirely and misreported `size` on blob
  blocks.)

## v0.1.1-fh-beta-10

### Changed

- Rebased onto upstream `v0.1.1` (`457f81a`): Pasteur mainnet activation scheduled
  (2026-08-25), upstream network/blocks-by-range improvements, and the new
  `bnb-chain/reth` pin (`c13b0986`) which streams finalization-appended (system-tx)
  receipts to the engine receipt-root task. The Firehose fork of the reth crates moved to
  `streamingfast/bnb-reth` branch `firehose/0.1.x-bsc` accordingly, with the same
  finalization-receipt streaming mirrored in the Firehose-traced engine execution path.
- Carries the beta-8 hard funding gate (replay funding never active under the Firehose
  inspector) and the beta-9 fast RocksDB `TransactionHashNumbers` healing.

Note: the block 106696194 pipeline divergence (deposit system tx short by one tx fee) is
NOT known to be fixed by this rebase — it reproduces with tracing disabled and is being
reported upstream.

## v0.1.0-fh-beta-7

### Added

- `FIREHOSE_DISABLED=true` kill-switch: skips tracer initialization entirely, so the node
  executes through the plain untraced path, byte-identical to un-instrumented reth-bsc. The
  firehose ExEx idles in no-op mode (still advancing `FinishedHeight` so the WAL prunes). Ops
  lever for isolating tracing-induced behavior — e.g. the block 106696194 deposit-value
  divergence under investigation.

## v0.1.0-fh-beta-3

### Fixed

- Panic during mainnet sync ("mismatch between call log and receipt log BlockIndex"): system
  transactions bypass the generic wrapper's per-transaction log accounting, so a log-bearing
  system tx (e.g. the validator deposit) left the block-wide log counter behind and the next
  system tx's call logs lagged its receipt logs. The chain executor now reports each system
  tx's committed log count and the inspector folds it into block-wide log indices.

## v0.1.0-fh-beta-2

### Fixed

- Panic during mainnet pipeline sync ("caller expected to be in transaction state"): BSC's
  internal consensus reads (validator-set / turn-length `eth_call`s) run on the same
  inspector-carrying EVM as real transactions and fired tracer hooks between transactions.
  These reads are now executed with Firehose tracing suspended — matching geth, which never
  traces them.

## v0.1.0-fh-beta-1

### Fixed

Parity with the geth-BSC Firehose reference (`streamingfast/go-ethereum`, `release/bnb-1.x-fh3.0`):

- Transaction fee (and EIP-4844 blob fee) balance changes are now credited to the consensus
  `SYSTEM_ADDRESS` (`0xffff…fffe`) instead of the block beneficiary, matching BSC's
  fee routing (`REASON_REWARD_TRANSACTION_FEE` / `REASON_REWARD_BLOB_FEE`).
- Parlia system transactions (deposit, slash, finality reward, validator-set update, genesis
  and Feynman initialization) are now traced as ordinary transaction traces at their actual
  execution point inside the end-of-block finalize step, instead of empty placeholder traces
  during body iteration with the real EVM work mis-attributed to a system-call window.
- The `distribute_incoming` sweep (SYSTEM_ADDRESS → validator, a direct non-EVM state write)
  now emits its two balance changes (`REASON_REWARD_TRANSACTION_FEE`), like geth's
  `BalanceDecrease/IncreaseBSCDistributeReward` hooks.
- System-contract code upgrades and the Prague history-storage deploy (direct non-EVM code
  installs) now emit code changes.

Known divergences from geth, by design or accepted: no gas-change events (dropped in Firehose
Ethereum tracer v5); system-tx sender nonce changes inside the EVM are emitted (geth suppresses
them under its backward-compatibility flag); the pre-Kepler two-step system-reward split emits a
single sweep pair here (modern blocks are identical).

## v0.1.0-fh-beta

### Added

- Firehose instrumentation for BSC: the `reth` crates are consumed from the
  [streamingfast/bnb-reth](https://github.com/streamingfast/bnb-reth) fork (branch
  `firehose/0.x-bsc`), which carries the `reth-firehose` crate and the engine-tree /
  pipeline tracing hooks. The `reth-bsc` binary initializes the process-wide tracer,
  wraps `BscEvmConfig` in `FirehoseEvmConfig` (pipeline path), and installs the
  Firehose ExEx — every validated block is emitted as a `FIRE BLOCK` line on stdout.
- `Dockerfile.sf` / `sf-release.yml`: Docker image build that bundles `fireeth` driving
  `reth-bsc` as its reader node, published to `ghcr.io/streamingfast/reth-bsc`.

### Fixed

- A fresh node (still at genesis) could never peer: the chain-spec `head()` helpers leave
  the head hash at the zero default, eth/69 status advertised that zero hash, and the
  handshake rejects zero blockhashes. The network head now falls back to the genesis hash.
