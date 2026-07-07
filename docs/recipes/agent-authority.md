# AI agent management

## The problem

AI agents now act on their own. They hold delegated authority to spend
money, call internal systems, move data, and sign on a principal's
behalf, at machine speed and in large numbers. That raises a governance
question every operator of agents has to answer: which agent took this
action, was it allowed to, was it inside its limits, and can I shut it
off right now? Today that authority is usually a bag of API keys and
config files, which are editable, copyable, and only as trustworthy as
whatever holds them.

Put the control plane on-chain and those questions get sharp answers. An
agent is registered under its own key with an explicit scope of
permissions, a spending budget, and an expiry. It acts by submitting
signed transactions, the contract checks each action against the
authority the owner granted, and revocation is a single transaction that
takes effect immediately and cannot be walked back by the agent.

An agent's authority is its cryptographic identity. The signatures that
carry that identity today (RSA, ECDSA) rest on math a quantum computer
can break. An adversary who could forge them could impersonate an agent
and wield everything it was trusted to do: drain its budget, invoke its
permissions, act as it. They could also forge the owner's grant, or
suppress a revocation. This is not a problem for later. Post-quantum
migration is mandated and underway now, agent keys already sit in logs,
transcripts, and on public ledgers where "harvest now, forge later"
collects them, and no one can rule out that a capable machine already
exists and is simply not announced.

So every registration, permission change, and agent action here is a
[post-quantum-signed transaction](index.md#why-fourier-contracts-are-quantum-proof-by-default).
`caller()` is the agent's post-quantum authenticated identity, so "which
agent did this, and was it authorized" cannot be forged even by an
adversary who already has a quantum computer.

## The contract

Source: `fourier/examples/agent_authority.fou` (compiles and runs on the
WaveLedger VM).

```fourier
// AgentAuthority: a management and authorization registry for AI agents.
//
// An owner registers an autonomous agent under the agent's own key, and
// grants it a scope of permissions, a spending budget, and an expiry.
// The agent acts by submitting post-quantum-signed transactions, so
// caller() is the agent's post-quantum authenticated identity. Every
// action is checked against the agent's granted authority and logged.
// The owner can retune permissions, top up or drain the budget, suspend,
// or permanently revoke the agent at any time.
contract AgentAuthority {
    storage registered: map[address, uint] @ 0;    // agent -> 1 once registered
    storage owner_of: map[address, address] @ 1;    // agent -> principal who controls it
    storage active: map[address, uint] @ 2;         // agent -> 1 if not revoked
    storage suspended: map[address, uint] @ 3;      // agent -> 1 if temporarily paused
    storage expiry: map[address, uint] @ 4;         // agent -> authority expiry (unix seconds)
    storage budget: map[address, uint] @ 5;         // agent -> remaining spend allowance
    storage perms: map[address, uint] @ 6;          // agent -> permission bitmask
    storage action_count: map[address, uint] @ 7;   // agent -> actions taken (audit)

    event AgentRegistered(agent: address, owner: address, perms: uint, budget: uint);
    event PermissionsSet(agent: address, perms: uint);
    event BudgetToppedUp(agent: address, amount: uint, new_budget: uint);
    event AgentSuspended(agent: address);
    event AgentResumed(agent: address);
    event AgentRevoked(agent: address);
    event AgentActed(agent: address, action_bit: uint, amount: uint, payload_hash: uint, remaining: uint);

    // A principal registers an agent under the agent's own key. The
    // caller becomes the owner and cannot be changed, so nobody can
    // hijack an agent identity someone else controls.
    pub fn register_agent(agent: address, mask: uint, initial_budget: uint, valid_seconds: uint) {
        require(registered[agent] == 0);
        registered[agent] = 1;
        owner_of[agent] = caller();
        active[agent] = 1;
        suspended[agent] = 0;
        perms[agent] = mask;
        budget[agent] = initial_budget;
        expiry[agent] = timestamp() + valid_seconds;
        emit AgentRegistered(agent, caller(), mask, initial_budget);
    }

    pub fn set_permissions(agent: address, mask: uint) {
        require(owner_of[agent] == caller());
        perms[agent] = mask;
        emit PermissionsSet(agent, mask);
    }

    pub fn top_up(agent: address, amount: uint) {
        require(owner_of[agent] == caller());
        budget[agent] = safe_add(budget[agent], amount);
        emit BudgetToppedUp(agent, amount, budget[agent]);
    }

    pub fn suspend(agent: address) {
        require(owner_of[agent] == caller());
        suspended[agent] = 1;
        emit AgentSuspended(agent);
    }

    pub fn resume(agent: address) {
        require(owner_of[agent] == caller());
        suspended[agent] = 0;
        emit AgentResumed(agent);
    }

    pub fn revoke(agent: address) {
        require(owner_of[agent] == caller());
        active[agent] = 0;
        emit AgentRevoked(agent);
    }

    // The agent itself calls this to perform an action. `action_bit` is a
    // single permission bit; the action succeeds only if the agent is
    // live, holds that permission, and has budget for `amount`. The work
    // itself is off-chain and anchored by `payload_hash`.
    pub fn act(action_bit: uint, amount: uint, payload_hash: uint) -> uint {
        let a: address = caller();
        require(registered[a] == 1);
        require(active[a] == 1);
        require(suspended[a] == 0);
        require(timestamp() <= expiry[a]);
        require(action_bit != 0);
        require((perms[a] & action_bit) == action_bit);
        let bal: uint = budget[a];
        require(bal >= amount);
        budget[a] = bal - amount;
        action_count[a] = action_count[a] + 1;
        emit AgentActed(a, action_bit, amount, payload_hash, budget[a]);
        return budget[a];
    }

    pub fn can_act(agent: address, action_bit: uint) -> uint {
        if action_bit == 0 {
            return 0;
        }
        if registered[agent] == 0 {
            return 0;
        }
        if active[agent] == 0 {
            return 0;
        }
        if suspended[agent] == 1 {
            return 0;
        }
        if timestamp() > expiry[agent] {
            return 0;
        }
        if (perms[agent] & action_bit) == action_bit {
            return 1;
        }
        return 0;
    }

    pub fn remaining_budget(agent: address) -> uint {
        return budget[agent];
    }

    pub fn get_owner(agent: address) -> address {
        return owner_of[agent];
    }
}
```

## How it works

### Storage

| Slot | Name | Type | Purpose |
|---|---|---|---|
| `0` | `registered` | `map[address, uint]` | agent → `1` once registered (permanent) |
| `1` | `owner_of` | `map[address, address]` | agent → principal who controls it |
| `2` | `active` | `map[address, uint]` | agent → `1` until revoked |
| `3` | `suspended` | `map[address, uint]` | agent → `1` while paused |
| `4` | `expiry` | `map[address, uint]` | agent → authority expiry (unix seconds) |
| `5` | `budget` | `map[address, uint]` | agent → remaining spend allowance |
| `6` | `perms` | `map[address, uint]` | agent → permission bitmask |
| `7` | `action_count` | `map[address, uint]` | agent → actions taken (audit) |

### Selector layout

| Selector | Function |
|---|---|
| `0x01` | `register_agent(address, uint, uint, uint)` |
| `0x02` | `set_permissions(address, uint)` |
| `0x03` | `top_up(address, uint)` |
| `0x04` | `suspend(address)` |
| `0x05` | `resume(address)` |
| `0x06` | `revoke(address)` |
| `0x07` | `act(uint, uint, uint) -> uint` |
| `0x08` | `can_act(address, uint) -> uint` |
| `0x09` | `remaining_budget(address) -> uint` |
| `0x0a` | `get_owner(address) -> uint` |

### Permissions are a capability bitmask

Each kind of action an agent may take is one bit. Assign meanings that
suit your system, for example bit `1` = read data, bit `2` = send
messages, bit `4` = spend funds. The `perms` mask is the set of bits an
agent holds. An action is allowed only when its bit is present:

```fourier
require(action_bit != 0);
require((perms[a] & action_bit) == action_bit);
```

Granting `5` (bits `1` and `4`) lets an agent read and spend but not
send messages. The owner can retune the mask at any time with
`set_permissions`, and the change takes effect on the agent's very next
action. (The parentheses matter: `&` binds looser than `==` in Fourier,
so the comparison must be wrapped.)

### Budgets, suspension, and instant revocation

Every action debits `budget`, and once it is spent the agent stops until
the owner tops it up. That caps the blast radius of a misbehaving or
hijacked agent to what you funded it with. `suspend` pauses an agent
without losing its grant; `resume` restores it. `revoke` sets `active`
to `0` permanently, and because the agent cannot flip its own flags, a
revoked agent is dead the instant the transaction lands.

### Identity is bound at registration

`register_agent` records the owner once and guards on
`registered[agent] == 0`, so an agent identity cannot be re-registered or
hijacked by anyone else later. Owner-only functions check
`owner_of[agent] == caller()`, and because `caller()` is post-quantum
authenticated, neither the owner's control nor the agent's own identity
can be spoofed.

### Verified end-to-end

Deploying the contract and driving an agent through its lifecycle on the
WaveLedger VM confirms the controls hold:

```text
registered; budget=1000 owner-set
act bit1 ok, remaining=700
permitted bit4 can_act     : True
unpermitted bit2 can_act   : False
unpermitted action blocked : True
over-budget blocked        : True
zero action_bit blocked    : True
non-owner suspend blocked  : True
suspended halts action     : True
resume + bit4 act ok, remaining=600
after top_up budget=1000
revoked halts action       : True
can_act after revoke       : False
re-register hijack blocked : True
expired agent act blocked  : True
END-TO-END AGENT AUTHORITY: PASS
```

## Driving it from your application

1. **Register an agent.** Generate a key for the agent, then call
   `register_agent(agent_address, perms_mask, budget, valid_seconds)`.
   You (the caller) become its owner.
2. **Hand the key to the agent.** The autonomous agent signs its own
   transactions with that key.
3. **Let it act.** The agent calls `act(action_bit, amount, payload_hash)`
   for each task, anchoring the off-chain work by its hash. The call
   reverts if the agent lacks the permission, is out of budget, is
   suspended, expired, or revoked.
4. **Govern it live.** `set_permissions` to widen or narrow scope,
   `top_up` to refill budget, `suspend` / `resume` to pause, and `revoke`
   to kill the agent permanently.
5. **Audit.** The `AgentActed` events form an immutable, per-agent record
   of every action and the budget left after it.

See the [ABI & calldata](../abi/index.md) reference for exact encoding
and the [Python](../compiler/python.md) compiler API to produce the
bytecode.

## Extending it

- **Real value, not just an allowance.** Have agents spend actual WAVE by
  attaching `callvalue()` and forwarding it, so the budget is a true
  on-chain spending limit rather than an internal counter.
- **Rate limiting.** Track a per-window action count and reject an agent
  that exceeds it, capping how fast a runaway agent can act.
- **Target allow-lists.** Restrict an agent to a set of approved
  counterparties or contract addresses, not just action types.
- **Delegation and sub-agents.** Let an agent register scoped sub-agents
  whose authority can never exceed the parent's, for hierarchical
  fleets.
- **Governed ownership.** Replace a single owner with a
  [multisig](../stdlib/multisig.md) so high-authority agents require a
  quorum to grant or revoke.
- **Off-chain agent keys.** For agents whose keys are not WaveLedger
  accounts, verify their post-quantum signatures with the
  [`verify_sig`](../quick-reference.md#crypto-scheme-ids) precompile,
  noting the v1 constraints on passing large signature blobs.
