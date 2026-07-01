# Tamper-proof file registry

## The problem

You have a file that matters and must not be silently altered: a signed
contract, a regulatory filing, a scientific dataset, a firmware image, a
legal exhibit, the weights of an AI model. Months or years later,
someone hands you a copy and says "this is the original."

**How do you know it wasn't changed?** And how does a third party, be it
an auditor, a court, or a customer, verify it *without* trusting you or
the storage vendor?

A blockchain solves this cheaply. You don't put the file on-chain (it's
public, and probably large). You anchor its **fingerprint**, a SHA3
hash, plus its ownership and full version history. Anyone holding a copy
can hash it themselves and compare; one changed byte produces a
completely different fingerprint, so tampering is instantly detectable.

The fingerprint proves a file is *unchanged*, but the record of **who
registered it, who may update it, and the history of versions** is only
trustworthy when those authorizations can't be forged, and today's
signatures won't hold forever. An attacker who could forge the owner's
signature could publish a malicious "new version" that looks officially
blessed: the exact software-supply-chain attack (xz, SolarWinds) that
keeps security teams awake. The signatures ordinarily guarding against
that rest on math a large quantum computer can break, and the public
keys needed to forge them are already on the ledger to be harvested now.
So registering and updating a file here are signed with a
[post-quantum scheme](index.md#why-fourier-contracts-are-quantum-proof-by-default):
only the real owner can publish a new version, and that holds even
against an adversary who already has a quantum computer.

!!! note "This protects integrity, not secrecy"
    The file stays wherever you keep it (S3, IPFS, a laptop). Only its
    hash is on-chain, so the registry reveals nothing about the
    contents. It only lets anyone *check* a copy against the anchored
    fingerprint.

## The contract

Source: `fourier/examples/file_registry.fou` (compiles on the WaveLedger
VM).

```fourier
// FileRegistry: a tamper-proof file system anchor.
// The file itself stays off-chain; only its SHA3 fingerprint is anchored
// here. Anyone can check whether a file they hold still matches the
// on-chain fingerprint, and the full version history is immutable.
contract FileRegistry {
    storage owner: map[uint, address] @ 0;                 // file_id -> owner
    storage current_hash: map[uint, uint] @ 1;             // file_id -> content fingerprint
    storage version: map[uint, uint] @ 2;                  // file_id -> current version
    storage registered: map[uint, uint] @ 3;               // file_id -> 1 if exists
    storage version_hash: map[uint, map[uint, uint]] @ 4;  // file_id -> version -> fingerprint
    storage updated_at: map[uint, uint] @ 5;               // file_id -> last update time

    event FileRegistered(id: uint, owner: address, content_hash: uint);
    event FileUpdated(id: uint, version: uint, content_hash: uint);
    event OwnershipTransferred(id: uint, new_owner: address);

    pub fn register_file(id: uint, content_hash: uint) {
        require(registered[id] == 0);
        registered[id] = 1;
        owner[id] = caller();
        current_hash[id] = content_hash;
        version[id] = 1;
        version_hash[id][1] = content_hash;
        updated_at[id] = timestamp();
        emit FileRegistered(id, caller(), content_hash);
    }

    pub fn update_file(id: uint, content_hash: uint) {
        require(owner[id] == caller());
        let v: uint = version[id] + 1;
        version[id] = v;
        current_hash[id] = content_hash;
        version_hash[id][v] = content_hash;
        updated_at[id] = timestamp();
        emit FileUpdated(id, v, content_hash);
    }

    // Returns 1 if the supplied fingerprint matches the current file.
    pub fn verify(id: uint, content_hash: uint) -> uint {
        if current_hash[id] == content_hash {
            return 1;
        }
        return 0;
    }

    pub fn transfer_ownership(id: uint, new_owner: address) {
        require(owner[id] == caller());
        owner[id] = new_owner;
        emit OwnershipTransferred(id, new_owner);
    }

    pub fn get_hash(id: uint) -> uint {
        return current_hash[id];
    }

    pub fn get_version(id: uint) -> uint {
        return version[id];
    }
}
```

## How it works

### Storage

| Slot | Name | Type | Purpose |
|---|---|---|---|
| `0` | `owner` | `map[uint, address]` | file id → who may update it |
| `1` | `current_hash` | `map[uint, uint]` | file id → SHA3 fingerprint of the latest version |
| `2` | `version` | `map[uint, uint]` | file id → current version number |
| `3` | `registered` | `map[uint, uint]` | file id → `1` if the id is taken |
| `4` | `version_hash` | `map[uint, map[uint, uint]]` | file id → version → fingerprint (the history) |
| `5` | `updated_at` | `map[uint, uint]` | file id → last update timestamp |

### Selector layout

| Selector | Function |
|---|---|
| `0x01` | `register_file(uint, uint)` |
| `0x02` | `update_file(uint, uint)` |
| `0x03` | `verify(uint, uint) -> uint` |
| `0x04` | `transfer_ownership(uint, address)` |
| `0x05` | `get_hash(uint) -> uint` |
| `0x06` | `get_version(uint) -> uint` |

### The fingerprint, not the file

The `content_hash` is a SHA3-256 hash of the file, computed off-chain by
your application. It fits in a single `uint` (32 bytes). Because a hash
is a one-way fingerprint:

- The registry never sees, stores, or leaks the file's contents.
- Any change to the file, even one bit, yields a different hash, so a
  mismatch against `current_hash` is proof of tampering.

### Immutable version history

`register_file` claims an id (guarded by `registered[id] == 0` so ids
can't be hijacked) and records version 1. Each `update_file` bumps the
version, overwrites `current_hash`, **and appends** the new fingerprint
to `version_hash[id][v]`. Old versions are never deleted. The result is
a permanent, ordered history (`version_hash[id][1]`,
`version_hash[id][2]`, and so on), each entry provable against the file
that produced it.

Both mutating functions are gated by `caller()` (`registered[id] == 0`
for the first claim, `owner[id] == caller()` thereafter), so only the
post-quantum-authenticated owner controls a file's lineage.

### Anyone can verify

`verify(id, hash)` returns `1` when a fingerprint matches the current
version, a one-call integrity check. A verifier's flow:

1. Receive a file claimed to be "document 42, latest."
2. Hash it locally with SHA3-256.
3. Call `verify(42, that_hash)`. `1` means authentic and current; `0`
   means altered, or not the latest version.

To check against a *specific* historical version instead, read
`version_hash[id][v]` directly.

## Driving it from your application

1. **Register:** hash your file, pick an id, call
   `register_file(id, hash)`. You become the owner.
2. **Update:** when the file changes, hash the new version and call
   `update_file(id, new_hash)`. The version bumps automatically and the
   old fingerprint stays in history.
3. **Distribute:** ship the file however you like. Recipients verify
   against the chain and never need your servers.
4. **Hand off:** `transfer_ownership(id, new_owner)` moves update rights,
   for example handing a document to its next custodian.

## Extending it

- **Multi-signer approval for updates.** Require an M-of-N sign-off
  (see the [threshold](../examples/threshold.md) and
  [multisig](../stdlib/multisig.md) patterns) before a new version takes
  effect, so no single owner key can push a poisoned release.
- **Revocation / freeze.** Add a `frozen[id]` flag that permanently
  blocks further updates once a file is finalized.
- **Directory index.** Layer a `map[address, array[uint]]` of "file ids
  owned by X" for enumeration.
- **Namespaced ids.** Derive `id` as `sha3(owner || path)` so
  human-meaningful paths map to registry ids without collisions.
