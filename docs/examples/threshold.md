# Threshold (M-of-N)

An M-of-N approval contract: an action executes only after at least
`M` distinct signers have voted for it. The stdlib ships a fuller
[`Multisig`](https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software/blob/main/fourier/stdlib/multisig.fou)
contract (`fourier/stdlib/multisig.fou`) that uses on-chain PQC
signature verification; this page covers a simpler "approval via
direct call" variant of the pattern.

Source: not shipped as a separate `.fou`. Use this page as a pattern
reference. The fuller stdlib version is at
`fourier/stdlib/multisig.fou`.

## Sketch

```fourier
contract Threshold {
    storage threshold: uint @ 0;
    storage signer_count: uint @ 1;
    storage is_signer: map[address, uint] @ 2;

    storage next_id: uint @ 3;
    storage target: map[uint, address] @ 4;
    storage value:  map[uint, uint] @ 5;
    storage executed: map[uint, uint] @ 6;
    storage approvals: map[uint, map[address, uint]] @ 7;
    storage approval_count: map[uint, uint] @ 8;

    event Proposed(id: uint, by: address, target: address);
    event Approved(id: uint, by: address);
    event Executed(id: uint);

    pub fn propose(t: address, v: uint) -> uint {
        require(is_signer[caller()] == 1);
        let id: uint = next_id;
        target[id] = t;
        value[id]  = v;
        executed[id] = 0;
        approval_count[id] = 0;
        next_id = id + 1;
        emit Proposed(id, caller(), t);
        return id;
    }

    pub fn approve(id: uint) -> uint {
        require(is_signer[caller()] == 1);
        require(executed[id] == 0);
        require(approvals[id][caller()] == 0);
        approvals[id][caller()] = 1;
        approval_count[id] = approval_count[id] + 1;
        emit Approved(id, caller());
        return 1;
    }

    pub fn execute(id: uint) -> uint {
        require(executed[id] == 0);
        require(approval_count[id] >= threshold);
        executed[id] = 1;
        let cd: bytes = pack_sel(0);     // empty calldata — value transfer only
        let ok: uint = call_b(target[id], cd, value[id], 200000);
        require(ok == 1);
        emit Executed(id);
        return 1;
    }
}
```

## Annotated source

### Storage

| Slot | Name | Purpose |
|---|---|---|
| `0` | `threshold` | M (required approvals) |
| `1` | `signer_count` | N (informational; populated separately) |
| `2` | `is_signer` | `address → 1` if signer |
| `3` | `next_id` | Auto-incrementing proposal id |
| `4` | `target` | `id → target address` |
| `5` | `value` | `id → WAVE to send` |
| `6` | `executed` | `id → 0/1` |
| `7` | `approvals` | `id → signer → 0/1` |
| `8` | `approval_count` | `id → count of approvers` |

Note: `is_signer` and `threshold` are not initialized in this
reference. An `init()` seeds them, or a separate `add_signer` /
`set_threshold` admin path gated by an owner provides the same effect.

### Selector layout

| Selector | Function |
|---|---|
| `0x01` | `propose(address, uint) -> uint` |
| `0x02` | `approve(uint) -> uint` |
| `0x03` | `execute(uint) -> uint` |

### Proposal flow

1. **propose**: a signer creates a proposal carrying `(target,
   value)`. Returns the proposal id.
2. **approve**: any signer (including the proposer) votes once.
   Re-approval reverts because `approvals[id][caller()] == 0` would
   fail.
3. **execute**: anyone can trigger execution once
   `approval_count[id] >= threshold`. The contract `call_b`s the
   target with empty calldata and forwards `value`. The success word
   is checked; failure reverts everything.

### Important subtleties

- `executed[id] = 1` is set **before** the external `call_b`. This
  is the checks-effects-interactions pattern: if the callee is
  malicious and attempts to re-enter `execute(id)`, the second
  invocation hits `require(executed[id] == 0)` and reverts. No
  separate reentrancy guard is needed.
- `approvals[id][caller()]` uses nested mapping syntax. The slot for
  `approvals[id][addr]` is
  `SHA3(addr_word || SHA3(id_word || slot_7_word))`. See
  [Storage / Nested mapping](../language/storage.md#nested-mapping).
- The empty-calldata `pack_sel(0)` produces a 1-byte `bytes` value
  containing the byte `0x00`. The callee's dispatcher does not match
  selector `0x00` (which is reserved for the deploy-time
  empty-calldata short-circuit), so this path succeeds only when the
  callee is an EOA or a contract whose code permits empty calldata.
  For real cross-contract invocation, pass the target function's
  selector.

## Upgrading to PQC

The fuller [`Multisig`](https://github.com/DosseyRichards/Fermi-Mining-ASIC-Software/blob/main/fourier/stdlib/multisig.fou)
contract in `fourier/stdlib/multisig.fou` uses the same pattern but
verifies each signer's signature on-chain via `verify_sig(...)` and a
PQC precompile. Use it when attestable signatures are required
rather than direct calls from signer EOAs.

The protocol is:

1. Signers compute a proposal hash off-chain.
2. Each signer ML-DSA-87-signs the hash.
3. Signers submit their signatures via `sign(id, sig)`.
4. The contract calls `verify_sig(1, pk, hash, sig)` and accepts on
   `return 1`.

This decouples the signing party from the EOA submitting the
transaction.

## Variants

- Add an `init()` that takes no parameters and seeds threshold and
  signers from constants.
- Make the signer set mutable via owner-gated `add_signer` /
  `remove_signer` calls.
- Extend `execute` to carry calldata as well: store
  `selector: map[uint, uint]` and `arg: map[uint, uint]`, then pass
  `pack_sel(selector[id], arg[id])` to `call_b`.
- For arbitrary calldata, use hash-commit/reveal: store
  `calldata_hash: map[uint, uint]` and require the executor to supply
  the calldata bytes at execute time (rehashed to verify).
