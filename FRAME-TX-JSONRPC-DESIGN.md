# EIP-8141 frame transactions — JSON-RPC representation (design)

This document specifies the JSON-RPC (OpenRPC) representation for EIP-8141 frame
transactions (transaction type `0x06`). It is the execution-apis companion to the
EIP: it lets block explorers and wallets read frame transactions, their receipts,
and their per-frame log attribution over the standard `eth_` methods
(`eth_getTransactionByHash`, `eth_getTransactionReceipt`, `eth_getBlockByHash`,
`eth_getLogs`, …).

Every field below is grounded in a working implementation of the type-`0x06`
JSON-RPC surface (`FrameTransactionForRpc`, `FrameForRpc`, `FrameSignatureForRpc`,
`ReceiptForRpc`, `FrameReceiptForRpc`) so the spec matches something that already
serializes on the wire.

## The gap

execution-apis models every typed transaction as a variant in the
`TransactionUnsigned` / `TransactionSigned` `oneOf` and every receipt through
`ReceiptInfo`. There is no variant for type `0x06`, so a frame transaction cannot
be represented at all: its `frames` and `signatures` lists, and the receipt's
`payer` and per-frame `frameReceipts`, have nowhere to live. Without this a frame
tx either fails schema validation or is silently coerced into a generic transaction
that drops its frame-specific fields.

## What changes

Two schema files, mirroring the existing 4844 / 7702 precedent:

- `src/schemas/transaction.yaml`
  - new `Frame` object schema (`[mode, flags, target, gasLimit, value, data]`)
  - new `FrameSignature` object schema (`[scheme, signer, msg, signature]`)
  - new `Transaction8141Unsigned` (type-`0x6` transaction body)
  - new `Transaction8141Signed`
  - add both to the `TransactionUnsigned` and `TransactionSigned` `oneOf` lists
- `src/schemas/receipt.yaml`
  - new `FrameReceipt` object schema (`[status, gasUsed, logs]`)
  - add `payer` and `frameReceipts` properties to `ReceiptInfo`

No method (`src/eth/*.yaml`) changes are required: the transaction and receipt
methods already reference `TransactionSigned` / `TransactionInfo` / `ReceiptInfo`,
so extending those component schemas propagates to every method automatically.

## The type-0x06 transaction object

EIP-8141 payload body (RLP):

```
[chain_id, nonce, sender, frames, signatures,
 max_priority_fee_per_gas, max_fee_per_gas, max_fee_per_blob_gas, blob_versioned_hashes]
frames     = [[mode, flags, target, gas_limit, value, data], ...]
signatures = [[scheme, signer, msg, signature], ...]
```

`Transaction8141Unsigned` fields:

| field                  | type               | notes |
|------------------------|--------------------|-------|
| `type`                 | `^0x6$`            | EIP `FRAME_TX_TYPE = 0x06` |
| `nonce`                | `uint`             | sender nonce |
| `maxPriorityFeePerGas` | `uint`             | EIP-1559 priority fee |
| `maxFeePerGas`         | `uint`             | EIP-1559 max fee |
| `maxFeePerBlobGas`     | `uint`             | EIP-4844; relevant only when `blobVersionedHashes` is non-empty |
| `gasPrice`             | `uint`             | deprecated effective-gas-price mirror, as on every other type |
| `frames`              | `array<Frame>`      | ordered frame list |
| `signatures`          | `array<FrameSignature>` | validated signatures available to the tx |
| `blobVersionedHashes` | `array<hash32>`     | EIP-4844 blob hashes |
| `chainId`             | `uint`              | |

Deliberately **absent** top-level fields, versus the 1559/4844 variants:

- **No `to`, `gas`, `value`, `input`.** These are per-frame in EIP-8141
  (`target`, `gasLimit`, `value`, `data` live on each `Frame`); there is no single
  top-level destination. This is the biggest structural departure from prior
  typed-tx variants.
- **No `accessList`.** EIP-8141 explicitly omits an access list ("No access list"
  rationale).
- **No `authorizationList`.** EIP-8141 explicitly omits the EIP-7702 auth list.

### `Frame` object

| field      | type      | notes |
|------------|-----------|-------|
| `mode`     | `byte`    | `0` DEFAULT (execute as `ENTRY_POINT`), `1` VERIFY (validation), `2` SENDER |
| `flags`    | `byte`    | bits 0-1 = APPROVE scope; bit 2 = atomic-batch (`ATOMIC_BATCH_FLAG = 0x4`) |
| `target`   | `address` | frame destination ("to") |
| `gasLimit` | `uint`    | max gas for the frame |
| `value`    | `uint`    | wei transferred from sender; non-zero only for SENDER frames |
| `data`     | `bytes`   | calldata for the frame's top-level call |

### `FrameSignature` object

| field       | type      | notes |
|-------------|-----------|-------|
| `scheme`    | `byte`    | verification scheme for the raw bytes |
| `signer`    | `address` | scheme-dependent signer metadata; a 20-byte address for `SECP256K1` and `P256` |
| `msg`       | `bytes`   | empty ⇒ canonical tx signature hash, else an explicit 32-byte digest |
| `signature` | `bytes`   | raw signature bytes, interpreted per `scheme` |

The raw `signature` bytes are surfaced for observability. The EIP's rule that
empty-`msg` signature bytes are *elided from the canonical signature hash* (and
that the EVM cannot introspect them) is an execution/consensus concern; it does not
restrict the JSON-RPC view, which reports the bytes as included in the block.

### Signed vs. unsigned — no outer `yParity`/`r`/`s`

Every existing typed-tx `…Signed` schema adds an outer secp256k1 signature
(`yParity`, `r`, `s`). **Frame transactions have no outer signature**: authorization
is carried entirely by the in-body `signatures` list. So `Transaction8141Signed`
is `allOf: [Transaction8141Unsigned]` with no added properties — it exists only to
populate the `TransactionSigned` `oneOf` symmetrically. Type `0x6` discriminates the
variant within the `oneOf`.

### `sender` / `from`

The EIP payload names the originator `sender`. In JSON-RPC the originator is
conventionally surfaced as `from`, which `TransactionInfo` already adds to every
mined transaction (and `GenericTransaction.from` covers request objects). To avoid a
redundant field, this design maps `sender` → `from` and does **not** add a separate
`sender` property. (See open question O1.)

## The receipt object

EIP-8141 `ReceiptPayload`:

```
[cumulative_gas_used, payer, [frame_receipt, ...]]
frame_receipt = [status, gas_used, logs]
```

Two new properties are added to `ReceiptInfo` (both present **only** for type
`0x06`, mirroring how `blobGasUsed` / `blobGasPrice` are 4844-only):

| field           | type                  | notes |
|-----------------|-----------------------|-------|
| `payer`         | `address`             | account that paid the fees; cannot be determined statically, hence recorded in the receipt |
| `frameReceipts` | `array<FrameReceipt>` | per-frame receipts, in frame order |

The `payer` rationale is taken directly from the EIP ("Payer in receipt"): the payer
is resolved during execution (by the frame that calls `APPROVE(APPROVE_PAYMENT)` /
`APPROVE(APPROVE_EXECUTION_AND_PAYMENT)`) and "the only way to provide this
information safely and efficiently over the JSON-RPC is to record this data in the
receipt object." Note it is therefore a **receipt** field, not a transaction-object
field — a frame transaction's JSON has no `payer` until it is executed.

### `FrameReceipt` object

| field      | type          | notes |
|------------|---------------|-------|
| `status`   | `byte`        | `0` failure, `1` success, `2` **skipped** (frame skipped by a failed atomic batch — new EIP-8141 code) |
| `gasUsed`  | `uint`        | total gas used by the frame, *not* accounting for refunds |
| `logs`     | `array<Log>`  | logs emitted by this frame (reuses the existing `Log` schema) |

The top-level receipt `status` remains `0`/`1` (the return code of the top-level
call); only per-frame `status` uses the new `0x2` skipped code.

## Log attribution

Per the EIP, "the transaction's logs, for the purposes of the block header
`logsBloom` and log indexing, are the concatenation of the `logs` fields of its
frame receipts, in frame order."

- The receipt's top-level `logs` array is the flat concatenation of all frame logs,
  in frame order, with the usual global `logIndex`. No schema change: existing
  tooling that reads `receipt.logs` / `eth_getLogs` keeps working unchanged.
- `frameReceipts[i].logs` is the per-frame subset, enabling explorers to attribute
  each log to the frame that produced it.
- Logs from frames unrolled by a failed atomic batch are discarded (their frame
  receipt retains its `status`/`gasUsed` but carries empty `logs`), so they never
  appear in either the flat array or the per-frame array.

## Open questions / forks for review

- **O1 — surface `sender` explicitly?** This design maps `sender` → `from`. A frame
  tx's sender is an explicit payload field, not derived from an outer signature, so
  one could argue for an explicit `sender` property on `Transaction8141Unsigned`.
  Recommendation: keep `from` only, to match convention and avoid duplication.
- **O2 — `target` requiredness / creation.** The RLP frame tuple always carries a
  `target`; frames do not perform contract creation, so `target` is modeled as a
  required, non-null `address`. (The reference implementation types it nullable but
  always serializes it.) Confirm frames can never be creations before finalizing.
- **O3 — `signer` generality.** `signer` is typed as `address` because the only
  defined schemes (`SECP256K1`, `P256`) use a 20-byte address. Future large-public-key
  schemes may need a more general `bytes`. Left as `address` to match today's
  implementation; revisit if/when a non-address scheme lands.
- **O4 — blob-carrying frame txs.** EIP-8141 allows non-empty `blobVersionedHashes`
  (blob-carrying frame txs, EIP-4844/7594 gossip). `maxFeePerBlobGas` and
  `blobVersionedHashes` are included here. The reference `FrameTransactionForRpc`
  currently extends the 1559 view (not the 4844 view), so it does not yet emit these
  two fields; the spec includes them for completeness and flags the implementation
  gap.
- **O5 — top-level `to`/`value` omission.** Some explorer tooling assumes every tx
  has a top-level `to`. This design omits it (per-frame only). If broad tooling
  compatibility is a hard requirement, an alternative is to surface a `null` `to`;
  recommendation is to omit and let consumers branch on `type == 0x6`.

## Validation status

Node.js is not installed in the authoring environment, so `npm run lint`
(`scripts/build.js` + `scripts/validate.js`) could not be executed here. The YAML
was validated for parse-correctness and `$ref`-target existence with a standalone
checker. The additions reuse only pre-existing base types (`byte`, `uint`,
`address`, `bytes`, `hash32`) and the existing `Log` schema, and mirror the 4844 /
7702 structures field-for-field, so they are expected to pass `lint` unchanged.
Running `npm run lint` in CI (or any Node environment) is the remaining verification
step.
