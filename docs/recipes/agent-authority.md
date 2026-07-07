# Agentic AI

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

An agent's authority is its cryptographic identity, and that raises the
stakes on forgery. A forged agent identity is worse than a stolen
password: it is an autonomous attacker operating with legitimate
authority, at machine speed, across as many actions as its budget allows,
before anyone reviews a single one. The signatures that carry that
identity today (RSA, ECDSA) rest on math a quantum computer can break. An
adversary who could forge them could impersonate an agent and wield
everything it was trusted to do, drain its budget, and invoke its
permissions. They could also forge the owner's grant or suppress a
revocation, so the off switch stops working exactly when you need it.

The stakes scale with what the agent is trusted to do. A support chatbot
with a small budget is a contained loss; an agent wired into a defense
network or a piece of critical infrastructure is not. Any of them whose
authority rests on RSA or ECDSA is impersonable by anyone holding a
cryptographically-relevant quantum computer, and the attacker does not
break the model, they become the agent. That is why the authentication
has to be post-quantum and the authority tightly scoped and instantly
revocable.

This is not a problem for later. Post-quantum migration is mandated and
underway now, agent keys already sit in logs, transcripts, and on public
ledgers where "harvest now, forge later" collects them, and no one can
rule out that a capable machine already exists and is simply not
announced. So every registration, permission change, and agent action
here is a
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
// action is checked against the agent's granted authority, appended to a
// queryable on-chain action log, and emitted as an event. The owner can
// retune permissions, top up or drain the budget, suspend, or
// permanently revoke the agent at any time.
contract AgentAuthority {
    storage registered: map[address, uint] @ 0;    // agent -> 1 once registered
    storage owner_of: map[address, address] @ 1;    // agent -> principal who controls it
    storage active: map[address, uint] @ 2;         // agent -> 1 if not revoked
    storage suspended: map[address, uint] @ 3;      // agent -> 1 if temporarily paused
    storage expiry: map[address, uint] @ 4;         // agent -> authority expiry (unix seconds)
    storage budget: map[address, uint] @ 5;         // agent -> remaining spend allowance
    storage perms: map[address, uint] @ 6;          // agent -> permission bitmask
    storage action_count: map[address, uint] @ 7;   // agent -> actions taken (also next log index)

    // On-chain action log, keyed by agent -> index (0 .. action_count-1).
    storage act_bit: map[address, map[uint, uint]] @ 8;      // index -> action bit
    storage act_amount: map[address, map[uint, uint]] @ 9;   // index -> amount spent
    storage act_payload: map[address, map[uint, uint]] @ 10; // index -> payload fingerprint
    storage act_time: map[address, map[uint, uint]] @ 11;    // index -> timestamp

    event AgentRegistered(agent: address, owner: address, perms: uint, budget: uint);
    event PermissionsSet(agent: address, perms: uint);
    event BudgetToppedUp(agent: address, amount: uint, new_budget: uint);
    event AgentSuspended(agent: address);
    event AgentResumed(agent: address);
    event AgentRevoked(agent: address);
    event AgentActed(agent: address, index: uint, action_bit: uint, amount: uint, payload_hash: uint, remaining: uint);

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
    // itself is off-chain and anchored by `payload_hash`. The action is
    // appended to the on-chain log at index action_count.
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
        let i: uint = action_count[a];
        act_bit[a][i] = action_bit;
        act_amount[a][i] = amount;
        act_payload[a][i] = payload_hash;
        act_time[a][i] = timestamp();
        action_count[a] = i + 1;
        emit AgentActed(a, i, action_bit, amount, payload_hash, budget[a]);
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

    // ── Queryable on-chain action log ────────────────────────────────
    // Any contract or caller can read an agent's past actions directly,
    // without an off-chain indexer. Valid indices are 0 .. count-1.
    pub fn action_count_of(agent: address) -> uint {
        return action_count[agent];
    }

    pub fn action_bit_at(agent: address, i: uint) -> uint {
        return act_bit[agent][i];
    }

    pub fn action_amount_at(agent: address, i: uint) -> uint {
        return act_amount[agent][i];
    }

    pub fn action_payload_at(agent: address, i: uint) -> uint {
        return act_payload[agent][i];
    }

    pub fn action_time_at(agent: address, i: uint) -> uint {
        return act_time[agent][i];
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
| `7` | `action_count` | `map[address, uint]` | agent → actions taken, and the next log index |
| `8` | `act_bit` | `map[address, map[uint, uint]]` | agent → index → action bit |
| `9` | `act_amount` | `map[address, map[uint, uint]]` | agent → index → amount spent |
| `10` | `act_payload` | `map[address, map[uint, uint]]` | agent → index → payload fingerprint |
| `11` | `act_time` | `map[address, map[uint, uint]]` | agent → index → timestamp |

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
| `0x0a` | `get_owner(address) -> address` |
| `0x0b` | `action_count_of(address) -> uint` |
| `0x0c` | `action_bit_at(address, uint) -> uint` |
| `0x0d` | `action_amount_at(address, uint) -> uint` |
| `0x0e` | `action_payload_at(address, uint) -> uint` |
| `0x0f` | `action_time_at(address, uint) -> uint` |

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

### The action log, on-chain and queryable

Every `act` call appends a record to the on-chain log at index
`action_count`, then bumps the count. Each record keeps the action bit,
the amount, the off-chain payload fingerprint, and the timestamp, and the
same fields go out as an `AgentActed` event. Any contract or caller can
walk an agent's history directly:

```text
n = action_count_of(agent)
for i in 0 .. n-1:
    action_bit_at(agent, i)
    action_amount_at(agent, i)
    action_payload_at(agent, i)
    action_time_at(agent, i)
```

The action's actual content stays off-chain; only its `payload_hash`
fingerprint is stored, the same commitment pattern as the
[file registry](file-registry.md) recipe.

### Storing the log on-chain versus emitting events: the tradeoff

This recipe does both, and the difference is worth understanding before
you copy it, because keeping the log in storage is not free.

Every logged action writes four extra storage slots on top of the budget
update. A fresh storage slot is the most expensive thing the VM charges
for, so an action goes from roughly one storage write to five, and the
gas cost rises accordingly. That matters for an agent acting at machine
speed. The log also grows the contract's live state without bound: every
node keeps that state forever, and nothing ever reclaims it.

The `AgentActed` event already records the identical fields permanently,
in the transaction log, which is far cheaper to write and does not bloat
the queryable state. So on-chain storage buys exactly one thing events do
not: another **contract** can read the history during its own execution,
and anyone can read it trustlessly without running an indexer.

The rule of thumb: keep the on-chain log only when on-chain logic needs
to inspect past actions, such as a rate limiter that looks back over a
window, or a dispute or slashing contract that must prove what an agent
did. If your readers are dashboards, auditors, or compliance tooling,
drop the four log maps and their getters and rely on the `AgentActed`
events. You get the same permanent, tamper-evident record for a fraction
of the cost, and you query it off-chain.

### Verified end-to-end

Deploying the contract and driving an agent through its lifecycle on the
WaveLedger VM confirms the controls hold and the log reads back:

```text
registered; budget=1000 owner-set
act bit1 ok, remaining=700
permitted bit4 can_act     : True
unpermitted bit2 can_act   : False
unpermitted action blocked : True
over-budget blocked        : True
non-owner suspend blocked  : True
suspended halts action     : True
action log queryable       : count=2  log[0]=(bit 1, amt 300)  log[1]=(bit 4, amt 100)
revoked halts action       : True
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
5. **Audit.** Read the on-chain log with `action_count_of` and the
   `action_*_at` getters when a contract needs it, or consume the
   `AgentActed` events off-chain for dashboards and compliance. Both are
   permanent.

See the [ABI & calldata](../abi/index.md) reference for exact encoding
and the [Python](../compiler/python.md) compiler API to produce the
bytecode.

## Extending it

- **Real value, not just an allowance.** Have agents spend actual WAVE by
  attaching `callvalue()` and forwarding it, so the budget is a true
  on-chain spending limit rather than an internal counter.
- **Rate limiting.** Use the on-chain log to reject an agent that has
  acted too many times in a window, capping how fast a runaway agent can
  act.
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
