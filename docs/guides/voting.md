# Quantum-proof voting

## The problem

Any group that votes — a co-op, a board, a union, a token community, a
municipal pilot — has to answer three awkward questions:

1. **Was every ballot cast by someone allowed to vote?**
2. **Did anyone vote twice?**
3. **Can we trust the final tally — that nobody added, dropped, or
   changed votes after the fact?**

Traditional systems answer these by asking you to *trust the operator*:
the server counting the votes, the admin with database access. A
blockchain replaces that trust with a public, tamper-proof record —
every ballot is a signed transaction, the counting is done by code
everyone can inspect, and the result is permanent.

But that record is only as trustworthy as the signatures authenticating
the ballots — and today's signatures (RSA, ECDSA) rest on math a large
quantum computer can break. A vote *is* a signature. Once those
signatures fall, an attacker can mint ballots that look completely
legitimate and forge them *retroactively* against an election held years
ago — and the public keys needed to do it are already on the ledger,
waiting to be harvested. For outcomes that must stand for decades —
constitutional changes, long-term mandates, endowment decisions — that's
not a distant worry but a permanent liability. So every ballot here is
signed with a [post-quantum scheme](index.md#why-fourier-contracts-are-quantum-proof-by-default)
a quantum computer can't forge, keeping the "was this a real, authorized
voter?" check and the recorded tally trustworthy into the quantum era.

## The contract

Source: `fourier/examples/quantum_vote.fou` (compiles and runs on the
WaveLedger VM).

```fourier
// QuantumVote — a tamper-proof voting workflow on WaveLedger.
// Every vote is a transaction signed with ML-DSA-87 (FIPS 204), so
// caller() is a post-quantum-authenticated identity and the tally is
// permanent and public.
contract QuantumVote {
    storage admin: address @ 0;
    storage proposal_count: uint @ 1;
    storage proposal_deadline: map[uint, uint] @ 2;         // id -> deadline (unix seconds)
    storage yes_votes: map[uint, uint] @ 3;                 // id -> yes tally
    storage no_votes: map[uint, uint] @ 4;                  // id -> no tally
    storage is_voter: map[address, uint] @ 5;              // address -> 1 if registered
    storage has_voted: map[uint, map[address, uint]] @ 6;  // id -> voter -> 1

    event VoterRegistered(voter: address);
    event ProposalCreated(id: uint, deadline: uint);
    event VoteCast(id: uint, voter: address, support: uint);

    // The deployer becomes the election administrator.
    fn init() {
        admin = caller();
    }

    pub fn register_voter(voter: address) {
        require(caller() == admin);
        is_voter[voter] = 1;
        emit VoterRegistered(voter);
    }

    pub fn create_proposal(voting_seconds: uint) -> uint {
        require(caller() == admin);
        let id: uint = proposal_count;
        proposal_deadline[id] = timestamp() + voting_seconds;
        proposal_count = id + 1;
        emit ProposalCreated(id, proposal_deadline[id]);
        return id;
    }

    pub fn cast_vote(id: uint, support: uint) {
        require(is_voter[caller()] == 1);
        require(timestamp() <= proposal_deadline[id]);
        require(has_voted[id][caller()] == 0);
        has_voted[id][caller()] = 1;
        if support == 1 {
            yes_votes[id] = yes_votes[id] + 1;
        } else {
            no_votes[id] = no_votes[id] + 1;
        }
        emit VoteCast(id, caller(), support);
    }

    pub fn get_yes(id: uint) -> uint {
        return yes_votes[id];
    }

    pub fn get_no(id: uint) -> uint {
        return no_votes[id];
    }
}
```

## How it works

### Storage

| Slot | Name | Type | Purpose |
|---|---|---|---|
| `0` | `admin` | `address` | Election administrator; set in `init` |
| `1` | `proposal_count` | `uint` | Next proposal id |
| `2` | `proposal_deadline` | `map[uint, uint]` | id → voting deadline (unix seconds) |
| `3` | `yes_votes` | `map[uint, uint]` | id → running "yes" tally |
| `4` | `no_votes` | `map[uint, uint]` | id → running "no" tally |
| `5` | `is_voter` | `map[address, uint]` | address → `1` if on the roll |
| `6` | `has_voted` | `map[uint, map[address, uint]]` | id → voter → `1` once used |

### Selector layout

| Selector | Function |
|---|---|
| `0x01` | `register_voter(address)` |
| `0x02` | `create_proposal(uint) -> uint` |
| `0x03` | `cast_vote(uint, uint)` |
| `0x04` | `get_yes(uint) -> uint` |
| `0x05` | `get_no(uint) -> uint` |

### The three questions, answered in code

**Who may vote** — the admin adds addresses to the roll. Because
`caller()` is post-quantum-authenticated, only the real admin passes the
gate:

```fourier
pub fn register_voter(voter: address) {
    require(caller() == admin);   // quantum-safe identity check
    is_voter[voter] = 1;
    emit VoterRegistered(voter);
}
```

**One vote each** — `cast_vote` checks the roll, the deadline, and the
per-proposal `has_voted` flag before counting. The double-vote guard is
the `has_voted[id][caller()] == 0` line:

```fourier
pub fn cast_vote(id: uint, support: uint) {
    require(is_voter[caller()] == 1);            // registered?
    require(timestamp() <= proposal_deadline[id]); // still open?
    require(has_voted[id][caller()] == 0);        // not already voted?
    has_voted[id][caller()] = 1;
    if support == 1 {
        yes_votes[id] = yes_votes[id] + 1;
    } else {
        no_votes[id] = no_votes[id] + 1;
    }
    emit VoteCast(id, caller(), support);
}
```

**A trustworthy tally** — `yes_votes` / `no_votes` are updated only
through `cast_vote`, and every change is a permanent event on a
post-quantum chain. There is no "edit the database" path.

### Verified end-to-end

Deploying this contract and running a full election on the WaveLedger VM
confirms the guards fire:

```text
proposal id = 0
double-vote blocked : True     # a registered voter voting twice reverts
non-voter blocked   : True     # an address not on the roll reverts
non-admin register  : True     # only the admin can register voters
tally yes/no = 1 / 1
END-TO-END VOTING: PASS
```

## Driving it from your application

Your app talks to the contract the same way it would any Fourier
contract — build calldata (a 1-byte selector followed by 32-byte
arguments) and submit it as a transaction. Conceptually:

1. **Deploy** the compiled contract. The deploying account becomes
   `admin`.
2. **Register** each eligible voter's address (`register_voter`).
3. **Open** a proposal with a voting window (`create_proposal(seconds)`).
4. Voters **cast** ballots (`cast_vote(id, 1)` for yes, `0` for no) —
   each signs their own transaction from their own wallet.
5. Anyone **reads** the result (`get_yes` / `get_no`), or reconstructs
   it independently from the `VoteCast` events.

See the [ABI & calldata](../abi/index.md) reference for exact encoding,
and the [Python](../compiler/python.md) compiler API to produce the
bytecode.

## Extending it

- **Weighted voting** — replace the `+ 1` increments with a
  `voter_weight[caller()]` lookup.
- **Secret ballots** — have voters submit a *commitment* (`sha3` of
  their choice + a nonce) during the window, then reveal after it
  closes. On-chain data is public, so a naive vote is visible; a
  commit-reveal scheme hides choices until the tally.
- **Quorum / thresholds** — add a `get_result(id)` that requires a
  minimum turnout before declaring an outcome.
- **Delegation** — add a `delegate(to)` map so a voter can assign their
  ballot to another registered address.
