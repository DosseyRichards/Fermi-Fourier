# Events

```fourier
event NAME(field1: TYPE, field2: TYPE, ...);
emit  NAME(expr, expr, ...);
```

Events are how contracts publish state changes to the chain log. The
VM `LOG` opcode records an entry with topics + data into the call
frame; on a successful tx, frame logs are merged into the receipt.

## Declaration

```fourier
event Transfer(from: address, to: address, amount: uint);
event Approval(owner: address, spender: address, amount: uint);
```

Field types are declared but **not** used for runtime encoding;
every argument word is 32 bytes regardless. Field types are part of
the signature hash that becomes topic 0 (see below).

## Signature hash (topic 0)

```python
sig_string = name + "(" + ",".join(type_name for _, type_name in params) + ")"
topic_0    = int.from_bytes(sha3_256(sig_string.encode()), "big")
```

For `event Transfer(from: address, to: address, amount: uint)`:

```text
sig_string = "Transfer(address,address,uint)"
topic_0    = SHA3-256(sig_string)
```

Type names use the Fourier keyword spelling: `uint`, `address`, `bool`,
`bytes`.

## Emit semantics

`emit NAME(arg1, ..., argN)` compiles to a `LOG<k>` opcode where
`k = min(1 + N, 4)`:

- The first up-to-3 args become indexed topics (`topic_1, topic_2, topic_3`).
- Any remaining args are written to memory and emitted as `data`.

So:

| Arg count `N` | Topics | Data |
|---|---|---|
| 0 | `[sig]` | empty |
| 1 | `[sig, arg0]` | empty |
| 2 | `[sig, arg0, arg1]` | empty |
| 3 | `[sig, arg0, arg1, arg2]` | empty |
| 4+ | `[sig, arg0, arg1, arg2]` | `arg3 \|\| arg4 \|\| ...` packed 32 bytes each |

Topic 0 is always the signature hash. Topics are 256-bit ints; data
is a byte string of `(N - 3) * 32` bytes when `N > 3`.

## Bytecode layout

For `emit Transfer(sender, to, amount)` (3 args → 4 topics, no data):

```text
PUSH amount         ; topic_3 (last indexed arg, deepest stack slot)
PUSH to             ; topic_2
PUSH sender         ; topic_1
PUSH topic_0_hash   ; topic_0 (signature)
PUSH 0              ; data length
PUSH 0              ; data offset (SCRATCH_A)
LOG 4
```

The LOG handler in `vm/machine.py`:

```python
offset = pop()
length = pop()
topics = [pop() for _ in range(n_topics)]    # [topic_0, topic_1, ...]
self._mem_expand(f, offset, length)
self._charge(f, 375 * n_topics + 8 * length)
f.logs.append(LogEntry(
    address=f.address,
    topics=topics,
    data=bytes(f.memory[offset:offset + length]),
))
```

## Constraints

- Maximum 4 topics. Topic 0 is reserved for the signature hash,
  leaving **up to 3 indexed arguments**.
- A `LOG` opcode is forbidden in a `STATICCALL` frame; emitting an
  event inside a `staticcall_b` callee reverts.
- Emit args are evaluated left-to-right. Side effects in earlier args
  happen before later args.

## Gas

Per `vm/opcodes.py` and `vm/machine.py`:

```text
base LOG       = 375
per topic      = 375  → 375 * n_topics
per data byte  = 8    → 8 * length
memory expansion to (offset, length) — standard EVM-like quadratic
```

## Reading logs off-chain

Each `LogEntry` becomes part of the transaction receipt:

```json
{
  "address": "<emitting contract address, hex>",
  "topics":  ["<topic_0 hash>", "<topic_1>", ...],
  "data":    "<hex>"
}
```

A subscriber filters by `topics[0]` (the signature hash). Compute the
expected topic_0 client-side:

```python
import hashlib
sig = "Transfer(address,address,uint)"
topic0 = hashlib.sha3_256(sig.encode()).hexdigest()
```

## Pattern: indexable args go first

Because the first 3 arguments become topics (cheaper to filter),
place query-by fields — sender/recipient addresses, ids — first, and
put bulky or rarely-queried fields last (these land in data):

```fourier
event Trade(
    maker: address,        // indexed: topic_1
    taker: address,        // indexed: topic_2
    pair_id: uint,         // indexed: topic_3
    qty: uint,             // data
    price: uint            // data
);
```
