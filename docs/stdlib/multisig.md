# Multisig (PQC)

Source: `fourier/stdlib/multisig.fou`.

An M-of-N approval contract where signers vote by submitting proposal
approvals. The shipped version uses direct on-chain approval rather
than off-chain signatures. The design supports extension with
`verify_sig(1, pk, hash, sig)` for full PQC attestation.

## Storage

| Slot | Name | Type | Purpose |
|---|---|---|---|
| `0` | `threshold` | `uint` | M (required signatures) |
| `1` | `signer_count` | `uint` | N (total signers) |
| `2` | `signers` | `map[uint, address]` | `i → signer address` |
| `3` | `is_signer` | `map[address, uint]` | `address → 1 if signer` |
| `4` | `next_proposal_id` | `uint` | Auto-incrementing id |
| `5` | `proposal_target` | `map[uint, address]` | |
| `6` | `proposal_value` | `map[uint, uint]` | |
| `7` | `proposal_executed` | `map[uint, uint]` | |
| `8` | `proposal_signatures` | `map[uint, map[address, uint]]` | |
| `9` | `proposal_sig_count` | `map[uint, uint]` | |

Reserves slots 0–9. Inheriting contracts must start their storage at
slot 10 or higher.

## Source

```fourier
contract Multisig {
    storage threshold: uint @ 0;
    storage signer_count: uint @ 1;
    storage signers: map[uint, address] @ 2;
    storage is_signer: map[address, uint] @ 3;

    storage next_proposal_id: uint @ 4;
    storage proposal_target: map[uint, address] @ 5;
    storage proposal_value: map[uint, uint] @ 6;
    storage proposal_executed: map[uint, uint] @ 7;
    storage proposal_signatures: map[uint, map[address, uint]] @ 8;
    storage proposal_sig_count: map[uint, uint] @ 9;

    event ProposalCreated(id: uint, target: address);
    event ProposalSigned(id: uint, signer: address);
    event ProposalExecuted(id: uint);

    pub fn propose(target: address, value: uint) -> uint {
        require(is_signer[caller()] == 1);
        let id: uint = next_proposal_id;
        proposal_target[id] = target;
        proposal_value[id] = value;
        proposal_executed[id] = 0;
        proposal_sig_count[id] = 0;
        next_proposal_id = id + 1;
        emit ProposalCreated(id, target);
        return id;
    }

    pub fn sign(id: uint) -> uint {
        require(is_signer[caller()] == 1);
        require(proposal_executed[id] == 0);
        require(proposal_signatures[id][caller()] == 0);
        proposal_signatures[id][caller()] = 1;
        proposal_sig_count[id] = proposal_sig_count[id] + 1;
        emit ProposalSigned(id, caller());
        return 1;
    }

    pub fn execute(id: uint) -> uint {
        require(proposal_executed[id] == 0);
        require(proposal_sig_count[id] >= threshold);
        proposal_executed[id] = 1;
        let cd: bytes = pack_sel(0);
        let ok: uint = call_b(
            proposal_target[id],
            cd,
            proposal_value[id],
            200000
        );
        require(ok == 1);
        emit ProposalExecuted(id);
        return 1;
    }
}
```

## Selectors

| Selector | Function |
|---|---|
| `0x01` | `propose(address, uint) -> uint` |
| `0x02` | `sign(uint) -> uint` |
| `0x03` | `execute(uint) -> uint` |

## Initialization

The shipped contract has **no `init`**. `threshold`, `signer_count`,
`signers`, and `is_signer` must be populated via an external
initialization step before any proposals are created. A real
deployment would add:

```fourier
fn init() {
    threshold = 2;
    signer_count = 3;
    signers[0] = 0x_signer_a;     // requires literal address constants
    signers[1] = 0x_signer_b;
    signers[2] = 0x_signer_c;
    is_signer[signers[0]] = 1;
    is_signer[signers[1]] = 1;
    is_signer[signers[2]] = 1;
}
```

Note: Fourier has no literal address syntax; embed literals as hex
`uint`s. The practical approach is an admin `add_signer(addr)`
function gated by an owner, called once per signer after deploy.

## Workflow

1. **Setup**: deploy the contract, set threshold and signer list
   (via admin paths added by the consumer; not shipped).
2. **Propose**: a signer calls `propose(target, value)` to create a
   pending proposal. The function returns the proposal id.
3. **Sign**: each approving signer calls `sign(id)`. The contract
   tracks unique signers (re-signing reverts) and increments the
   count.
4. **Execute**: once `proposal_sig_count[id] >= threshold`, any
   caller can invoke `execute(id)`. The contract sets the executed
   flag, sends `value` WAVE to `target` via `call_b`, and emits an
   event.

The executed flag is set **before** the external call —
checks-effects-interactions. A reentrant `execute(id)` from the
callee fails the `executed == 0` check.

## Limitations of the shipped version

- **No on-chain signature verification.** Signers approve by calling
  `sign(id)` directly from their EOA. To use PQC signatures, the
  contract must accept a `(pk, sig)` pair on each `sign(id, pk, sig)`
  call and verify with `verify_sig(1, pk, proposal_hash, sig)`.
- **No calldata in proposals.** `execute` calls the target with
  empty calldata (`pack_sel(0)`), so only plain WAVE transfers are
  supported. Arbitrary calls require adding `proposal_selector` and
  `proposal_arg` mappings, then passing
  `pack_sel(proposal_selector[id], proposal_arg[id])`.
- **Signer set is immutable after init.** Add admin paths for signer
  rotation.

The fuller [Timelock](timelock.md) contract supports calldata in
proposals; apply that approach to extend `Multisig`.

## Upgrade to full PQC

The pattern for verifying signatures on-chain:

```fourier
pub fn sign(id: uint, pk: bytes, sig: bytes) -> uint {
    require(is_signer[caller()] == 1);
    require(proposal_executed[id] == 0);
    require(proposal_signatures[id][caller()] == 0);

    // Reconstruct the proposal hash off-chain in your client; pass
    // it via storage or recompute it on-chain from proposal fields.
    let hash: bytes = ...;  // build with pack_sel + serialization
    let valid: uint = verify_sig(1, pk, hash, sig);   // ML-DSA-87
    require(valid == 1);

    proposal_signatures[id][caller()] = 1;
    proposal_sig_count[id] = proposal_sig_count[id] + 1;
    emit ProposalSigned(id, caller());
    return 1;
}
```

See [Expressions / PQC signature verify](../language/expressions.md#pqc-signature-verify).
