# Nuclear command authorization

!!! note "What this models"
    This recipe is the **authorization-control layer**: the command-
    authority quorum and two-person rule that decide whether an order is
    valid and prove who authorized it. It contains nothing operational.
    The order body stays off-chain and is anchored only by its
    fingerprint. Real command and control is far more than a contract;
    what generalizes here is the high-assurance authorization pattern and
    its need for unforgeable authentication.

## The problem

Nuclear command and control rests on one question above all others: is
this order genuine, and did it truly come from the people authorized to
give it? The controls built around that question are famously strict. No
single person can act alone (the two-person rule), an order must be
authenticated as coming from the legitimate command authority, and any
authorized officer can order a stand-down. What makes all of it work is
cryptographic authentication. An order is trusted because its
authorization can be proven, and a forged authorization is assumed to be
impossible.

That assumption is what a quantum computer threatens. The signatures that
authenticate today's systems (RSA, ECDSA) rest on math a quantum computer
can break. An adversary who could forge those signatures could
authenticate a fraudulent order, impersonate an officer's confirmation,
or suppress a legitimate stand-down. The pressure is immediate. National
security systems are already mandated to move to
post-quantum cryptography, because these platforms stay in service for
decades (some strategic systems have run for half a century) and because
"harvest now, forge later" means authentication material intercepted
today can be stored and broken once a capable machine is in reach. No one
can rule out that such a machine already exists and is simply not being
advertised.

This recipe models the authorization layer of that problem. A threshold
of distinct, [post-quantum-authenticated](index.md#why-fourier-contracts-are-quantum-proof-by-default)
officers must each confirm an order before it is valid, any one of them
can stand it down, and the order body stays off-chain, anchored only by
its fingerprint. `caller()` is a post-quantum authenticated officer
identity, so "who authorized this" cannot be forged even by an adversary
who already has a quantum computer.

## The contract

Source: `fourier/examples/command_authority.fou` (compiles and runs on the
WaveLedger VM).

```fourier
// NuclearCommandAuthority: a two-person-rule authorization workflow.
//
// This models the COMMAND-AUTHORITY layer of a high-assurance system:
// an action order is only valid once a threshold of distinct, authorized
// officers have each independently confirmed it, and any single officer
// can order a stand-down. It authenticates and authorizes orders. It is
// not a weapon system and carries no operational content; the order body
// lives off-chain and is anchored here only by its SHA3 fingerprint.
//
// Every roster change, confirmation, stand-down, and execution is a
// post-quantum-signed transaction, so caller() is a post-quantum
// authenticated officer identity.
contract NuclearCommandAuthority {
    storage commander: address @ 0;                 // authority who manages the roster
    storage threshold: uint @ 1;                    // distinct confirmations required to authorize
    storage is_officer: map[address, uint] @ 2;     // officer -> 1 if on the roster
    storage order_count: uint @ 3;

    storage order_hash: map[uint, uint] @ 4;        // id -> SHA3 fingerprint of the off-chain order
    storage order_proposer: map[uint, address] @ 5;
    storage order_deadline: map[uint, uint] @ 6;    // id -> expiry (unix seconds)
    storage order_confirms: map[uint, uint] @ 7;    // id -> distinct confirmations so far
    storage order_executed: map[uint, uint] @ 8;    // id -> 1 once executed
    storage order_aborted: map[uint, uint] @ 9;     // id -> 1 once stood down
    storage confirmed_by: map[uint, map[address, uint]] @ 10;  // id -> officer -> 1

    event OfficerAdded(officer: address);
    event OfficerRemoved(officer: address);
    event ThresholdSet(threshold: uint);
    event OrderIssued(id: uint, proposer: address, order_hash: uint);
    event OrderConfirmed(id: uint, officer: address, confirms: uint);
    event OrderAborted(id: uint, officer: address);
    event OrderExecuted(id: uint, order_hash: uint);

    // The deployer is the commander: they manage the officer roster and
    // set the confirmation threshold, but cannot authorize an order alone.
    fn init() {
        commander = caller();
    }

    pub fn add_officer(officer: address) {
        require(caller() == commander);
        is_officer[officer] = 1;
        emit OfficerAdded(officer);
    }

    pub fn remove_officer(officer: address) {
        require(caller() == commander);
        is_officer[officer] = 0;
        emit OfficerRemoved(officer);
    }

    pub fn set_threshold(m: uint) {
        require(caller() == commander);
        require(m >= 1);
        threshold = m;
        emit ThresholdSet(m);
    }

    // An authorized officer proposes an action order. The order body is
    // off-chain; only its SHA3 fingerprint is anchored. The proposer
    // counts as the first confirmation.
    pub fn issue_order(content_hash: uint, valid_seconds: uint) -> uint {
        require(is_officer[caller()] == 1);
        require(threshold >= 1);
        let id: uint = order_count;
        order_hash[id] = content_hash;
        order_proposer[id] = caller();
        order_deadline[id] = timestamp() + valid_seconds;
        order_confirms[id] = 1;
        confirmed_by[id][caller()] = 1;
        order_count = id + 1;
        emit OrderIssued(id, caller(), content_hash);
        return id;
    }

    // A distinct authorized officer independently confirms the order.
    pub fn confirm_order(id: uint) {
        require(is_officer[caller()] == 1);
        require(order_executed[id] == 0);
        require(order_aborted[id] == 0);
        require(timestamp() <= order_deadline[id]);
        require(confirmed_by[id][caller()] == 0);
        confirmed_by[id][caller()] = 1;
        order_confirms[id] = order_confirms[id] + 1;
        emit OrderConfirmed(id, caller(), order_confirms[id]);
    }

    // Fail-safe: any single authorized officer can stand the order down
    // before it executes. Going takes a quorum; stopping takes one.
    pub fn abort_order(id: uint) {
        require(is_officer[caller()] == 1);
        require(order_executed[id] == 0);
        order_aborted[id] = 1;
        emit OrderAborted(id, caller());
    }

    // Execute only when a full quorum has confirmed, the order is live,
    // and nobody has stood it down.
    pub fn execute_order(id: uint) {
        require(order_executed[id] == 0);
        require(order_aborted[id] == 0);
        require(timestamp() <= order_deadline[id]);
        require(order_confirms[id] >= threshold);
        order_executed[id] = 1;
        emit OrderExecuted(id, order_hash[id]);
    }

    pub fn confirmations(id: uint) -> uint {
        return order_confirms[id];
    }

    pub fn is_authorized(id: uint) -> uint {
        if order_aborted[id] == 1 {
            return 0;
        }
        if order_executed[id] == 1 {
            return 0;
        }
        if timestamp() > order_deadline[id] {
            return 0;
        }
        if order_confirms[id] >= threshold {
            return 1;
        }
        return 0;
    }

    pub fn get_order_hash(id: uint) -> uint {
        return order_hash[id];
    }
}
```

## How it works

### Storage

| Slot | Name | Type | Purpose |
|---|---|---|---|
| `0` | `commander` | `address` | Manages the roster and threshold; cannot authorize alone |
| `1` | `threshold` | `uint` | Distinct confirmations required to authorize an order |
| `2` | `is_officer` | `map[address, uint]` | officer → `1` if on the roster |
| `3` | `order_count` | `uint` | Next order id |
| `4` | `order_hash` | `map[uint, uint]` | id → SHA3 fingerprint of the off-chain order |
| `5` | `order_proposer` | `map[uint, address]` | id → officer who issued it |
| `6` | `order_deadline` | `map[uint, uint]` | id → expiry (unix seconds) |
| `7` | `order_confirms` | `map[uint, uint]` | id → distinct confirmations so far |
| `8` | `order_executed` | `map[uint, uint]` | id → `1` once executed |
| `9` | `order_aborted` | `map[uint, uint]` | id → `1` once stood down |
| `10` | `confirmed_by` | `map[uint, map[address, uint]]` | id → officer → `1` |

### Selector layout

| Selector | Function |
|---|---|
| `0x01` | `add_officer(address)` |
| `0x02` | `remove_officer(address)` |
| `0x03` | `set_threshold(uint)` |
| `0x04` | `issue_order(uint, uint) -> uint` |
| `0x05` | `confirm_order(uint)` |
| `0x06` | `abort_order(uint)` |
| `0x07` | `execute_order(uint)` |
| `0x08` | `confirmations(uint) -> uint` |
| `0x09` | `is_authorized(uint) -> uint` |
| `0x0a` | `get_order_hash(uint) -> uint` |

### The two-person rule

An order is authorized only when `order_confirms[id]` reaches
`threshold`, and each officer can count only once. The `confirmed_by`
nested map enforces distinctness:

```fourier
require(confirmed_by[id][caller()] == 0);   // this officer hasn't confirmed yet
confirmed_by[id][caller()] = 1;
order_confirms[id] = order_confirms[id] + 1;
```

Because `caller()` is post-quantum authenticated, one compromised or
forged identity cannot stand in for a second officer, and the commander
who runs the roster cannot manufacture a quorum alone.

### Going takes a quorum; stopping takes one

The safety asymmetry is deliberate. `execute_order` requires a full
`threshold` of confirmations, but `abort_order` needs a single
authorized officer and permanently voids the order:

```fourier
pub fn abort_order(id: uint) {
    require(is_officer[caller()] == 1);
    require(order_executed[id] == 0);
    order_aborted[id] = 1;
    emit OrderAborted(id, caller());
}
```

Once `order_aborted[id] == 1`, both `is_authorized` and `execute_order`
refuse the order. It is easy to stop and hard to go, which is the correct
default for a high-consequence action.

### Time-boxed and anchored

Every order carries a deadline. After it passes, confirmation and
execution both revert, so a stale order cannot be resurrected later. The
order's content lives off-chain; `order_hash` anchors its SHA3 fingerprint
so the authorization is bound to one exact order and nothing else.

### Verified end-to-end

Deploying the contract and running a full authorization cycle on the
WaveLedger VM confirms the controls hold:

```text
order id: 0
after issue: confirms=1 authorized=0 (need 2)
outsider confirm blocked   : True
proposer double-confirm     : True
execute below quorum        : True
after 2nd confirm: confirms=2 authorized=1
executed once               : True
re-execute blocked          : True
single-officer stand-down   : True
execute after abort blocked : True
confirm after expiry block  : True
END-TO-END COMMAND AUTHORITY: PASS
```

## Driving it from your application

1. **Deploy.** The deployer becomes the `commander`.
2. **Set the roster.** `add_officer(address)` for each authorized officer,
   then `set_threshold(m)` for the required number of independent
   confirmations (2 for a classic two-person rule, higher for a wider
   quorum).
3. **Issue an order.** An officer hashes the off-chain order and calls
   `issue_order(hash, valid_seconds)`. They count as the first
   confirmation.
4. **Confirm.** Each additional officer independently calls
   `confirm_order(id)` from their own credential. `is_authorized(id)`
   turns to `1` when the quorum is reached.
5. **Execute or stand down.** `execute_order(id)` finalizes an authorized,
   live order. Any officer can call `abort_order(id)` first to void it.

See the [ABI & calldata](../abi/index.md) reference for exact encoding
and the [Python](../compiler/python.md) compiler API to produce the
bytecode.

## Extending it

- **Role-separated quorums.** Require confirmations from distinct roles or
  chains of command (for example one officer from each of two commands),
  not just any two officers, by tagging each officer with a role and
  counting per role.
- **Mandatory cool-off.** Add a delay between reaching authorization and
  allowing `execute_order`, during which a stand-down is still possible,
  using the [Timelock](../stdlib/timelock.md) pattern.
- **Off-chain or air-gapped signers.** For officers whose keys are not
  WaveLedger accounts (hardware tokens, sealed authenticators), verify
  their post-quantum signatures with the
  [`verify_sig`](../quick-reference.md#crypto-scheme-ids) precompile,
  noting the v1 constraints on passing large signature blobs.
- **Governed roster.** Replace the single `commander` with a
  [multisig](../stdlib/multisig.md) so roster changes themselves need a
  quorum.
- **Tamper-evident audit stream.** The `OrderIssued` / `OrderConfirmed` /
  `OrderAborted` / `OrderExecuted` events already form an immutable log;
  mirror them to an external monitor for real-time oversight.
