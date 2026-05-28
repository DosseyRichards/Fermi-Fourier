# Guestbook

An append-only log of short messages, indexed by author. Demonstrates
storage arrays, mappings as secondary indices, and event emission.

Source: not shipped as a separate `.fou` in v1 — this page walks the
pattern with the primitives that exist (`array[T]`, `len/push/pop`,
`emit`, `map[address, uint]`).

## Sketch

```fourier
contract Guestbook {
    storage owner:        address @ 0;
    storage entries:      array[uint] @ 1;          // packed message hashes
    storage authors:      map[uint, address] @ 2;   // index → author
    storage entry_count_by_author: map[address, uint] @ 3;

    event Signed(idx: uint, by: address, msg_hash: uint);

    fn init() {
        owner = caller();
    }

    pub fn sign(msg_hash: uint) -> uint {
        let idx: uint = push(entries, msg_hash);   // returns new length
        // idx is the new length; the actual position is idx - 1
        let position: uint = idx - 1;
        authors[position] = caller();
        entry_count_by_author[caller()] = entry_count_by_author[caller()] + 1;
        emit Signed(position, caller(), msg_hash);
        return position;
    }

    pub fn count() -> uint {
        return len(entries);
    }

    pub fn get(idx: uint) -> uint {
        return entries[idx];
    }

    pub fn author_of(idx: uint) -> address {
        return authors[idx];
    }

    pub fn count_by(addr: address) -> uint {
        return entry_count_by_author[addr];
    }

    pub fn clear() {
        require(caller() == owner);
        while len(entries) > 0 {
            let popped: uint = pop(entries);
        }
        // Note: this drops the entries[] vector but `authors` and
        // `entry_count_by_author` keep their stale entries. A real impl
        // would either iterate and clear, or use a generation counter.
    }
}
```

## Walkthrough

### Storage

| Slot | Name | Type | Purpose |
|---|---|---|---|
| `0` | `owner` | `address` | Set in init; gates `clear` |
| `1` | `entries` | `array[uint]` | The append-only log. Length at slot 1; element `i` at `SHA3(slot1_word) + i` |
| `2` | `authors` | `map[uint, address]` | Secondary index: `position → who signed it` |
| `3` | `entry_count_by_author` | `map[address, uint]` | How many times each address has signed |

The "message" is a single `uint` (use a SHA3-256 hash of the off-chain
text). Fourier has no string type; storing variable-length data
on-chain would require multiple slots — out of scope for this sketch.

### Selector layout

| Selector | Function |
|---|---|
| `0x01` | `sign(uint) -> uint` |
| `0x02` | `count() -> uint` |
| `0x03` | `get(uint) -> uint` |
| `0x04` | `author_of(uint) -> address` |
| `0x05` | `count_by(address) -> uint` |
| `0x06` | `clear()` |

### `sign(msg_hash)`

```fourier
let idx: uint = push(entries, msg_hash);
let position: uint = idx - 1;
authors[position] = caller();
entry_count_by_author[caller()] = entry_count_by_author[caller()] + 1;
emit Signed(position, caller(), msg_hash);
return position;
```

`push(entries, msg_hash)` returns the **new length** (after the
push), so the position of the new entry is `length - 1`. The codegen
for `push`:

```text
SLOAD slot=1                     ; old_len
DUP 1                             ; old_len, old_len
PUSH 1, MSTORE 0                  ; write slot-base 1 to SCRATCH_A
PUSH 32, PUSH 0, SHA3             ; base_hash = SHA3(1_word)
DUP 3, ADD                        ; slot = base_hash + old_len
PUSH msg_hash                     ; new value
SWAP 1, SSTORE                    ; store entries[old_len] = msg_hash
PUSH 1, ADD                       ; new_len = old_len + 1
PUSH 1, SSTORE                    ; persist new length
PUSH 1, ADD                       ; return value = new_len = old_len + 1
```

The event emits a `LOG3` (sig + 2 indexed: `idx`, `by` — `msg_hash`
is the data arg).

Wait — actually, with 3 args, all 3 become topics. Let me correct:
`Signed(idx, by, msg_hash)` has 3 args, so:

- Topic 0: signature hash of `"Signed(uint,address,uint)"`
- Topic 1: `idx`
- Topic 2: `by`
- Topic 3: `msg_hash`

No data. See [Events / Emit semantics](../language/events.md#emit-semantics).

### `count()`, `get(idx)`, `author_of(idx)`, `count_by(addr)`

Straight read paths. `count()` is a single `SLOAD slot=1`.
`get(idx)` derives `entries[idx]`'s slot via `SHA3(slot1_word) + idx`
and `SLOAD`s.

### `clear()`

```fourier
require(caller() == owner);
while len(entries) > 0 {
    let popped: uint = pop(entries);
}
```

Loop `pop`s entries one at a time. Each `pop` does:

1. `SLOAD` length, revert if 0.
2. Decrement length, `SSTORE` new length.
3. Read element at `base_hash + new_len`.
4. Clear that element slot (writes 0, which deletes the storage entry).

For a large array, this can easily exhaust gas — `pop` is 4 SSTOREs
plus an SHA3 per iteration. A real implementation should bound the
loop:

```fourier
let mut_clear: uint = 0;
while mut_clear < 50 && len(entries) > 0 {
    let _: uint = pop(entries);
    mut_clear = mut_clear + 1;
}
```

Note: Fourier has no `mut` keyword — locals are always mutable. The
above is just for clarity.

### The `authors` mapping is left stale

After `clear()`, the `entries` array length is 0, but
`authors[N]` for `N` less than the original length still holds
the old signer addresses (with the noted exception: `pop` clears the
**array element** slot, not the `authors` mapping). A real impl would
either iterate clear `authors[i] = 0` for each `i`, or implement a
"generation counter" pattern:

```fourier
storage generation: uint @ 4;
storage authors_by_gen: map[uint, map[uint, address]] @ 5;
```

Then `clear()` only needs `generation = generation + 1` to invalidate
the entire mapping at once.

## What to try next

- Add the generation-counter pattern above and use it instead of
  iterating in `clear`.
- Make entries cap themselves at N (oldest gets popped when a new
  one is pushed beyond N).
- Add `event Cleared(by: address, removed_count: uint)`.
- Replace `entry_count_by_author` with computed sums (walk and count)
  to save a slot, if writes are more common than reads of that count.
