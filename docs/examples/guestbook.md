# Guestbook

An append-only log of short messages indexed by author. Exercises
storage arrays, mappings as secondary indices, and event emission.

Source: not shipped as a separate `.fou`. This page walks the
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

## Annotated source

### Storage

| Slot | Name | Type | Purpose |
|---|---|---|---|
| `0` | `owner` | `address` | Set in init; gates `clear` |
| `1` | `entries` | `array[uint]` | The append-only log. Length at slot 1; element `i` at `SHA3(slot1_word) + i` |
| `2` | `authors` | `map[uint, address]` | Secondary index: `position → who signed it` |
| `3` | `entry_count_by_author` | `map[address, uint]` | How many times each address has signed |

The "message" is a single `uint` — a SHA3-256 hash of the off-chain
text. Fourier has no string type; storing variable-length data
on-chain requires multiple slots and is out of scope for this sketch.

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

The event emits a `LOG4`. With 3 arguments, all three become topics:

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

For a large array, this can exhaust gas — `pop` is four SSTOREs
plus an SHA3 per iteration. Production implementations should bound
the loop:

```fourier
let mut_clear: uint = 0;
while mut_clear < 50 && len(entries) > 0 {
    let _: uint = pop(entries);
    mut_clear = mut_clear + 1;
}
```

Note: Fourier has no `mut` keyword; locals are always mutable. The
prefix here exists for clarity only.

### The `authors` mapping is left stale

After `clear()`, the `entries` array length is 0, but
`authors[N]` for `N` less than the original length still holds
the old signer addresses. `pop` clears the **array element** slot,
not the `authors` mapping. A production implementation would either
iterate `authors[i] = 0` for each `i`, or use a generation-counter
pattern:

```fourier
storage generation: uint @ 4;
storage authors_by_gen: map[uint, map[uint, address]] @ 5;
```

With this layout, `clear()` only needs `generation = generation + 1`
to invalidate the entire mapping at once.

## Variants

- Adopt the generation-counter pattern above in place of iteration in
  `clear`.
- Cap entries at N by popping the oldest when a new one is pushed
  beyond N.
- Add `event Cleared(by: address, removed_count: uint)`.
- Replace `entry_count_by_author` with a computed sum (walk and
  count) to save a slot when writes outweigh reads.
