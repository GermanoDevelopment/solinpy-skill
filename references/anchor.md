# Anchor programs with SolInPy

`solinpy.anchor.program.Program` reads an Anchor IDL at runtime and exposes each
instruction as a dynamic Python method. No codegen, no `.pyi` — attribute access is
resolved against the IDL on every call.

## Loading

```python
from solinpy.anchor.program import Program

program = Program.load("target/idl/my_program.json", client)          # path or dict
program = Program.load(idl_dict, client, program_id="Prog1111...")
```

Without an explicit `program_id`, it reads `idl["metadata"]["address"]` and raises
`ValueError` if absent.

`Program(idl, program_id, client)` is the direct constructor. `client` may be either
`SolanaRPCClient` or `SolanaAsyncRPCClient` — async is detected by inspecting
`client.send_transaction` for a coroutine function, and the returned call is
awaitable accordingly.

## Two namespaces

```python
ix  = program.instruction.initialize(42, accounts={...})            # -> Instruction
sig = program.rpc.initialize(42, accounts={...}, signers=[kp])      # builds, signs, sends
```

- `program.instruction.<name>` builds and returns a `solders.Instruction` — use this
  to compose multi-instruction transactions, and it is the reliable path today.
- `program.rpc.<name>` also sends. It requires `signers=[keypair, ...]`; the first
  signer pays. Omitting it raises `ValueError`.

Instruction names resolve by exact match **or** snake_case match, so both
`program.rpc.initializeVault(...)` and `program.rpc.initialize_vault(...)` work.
An unknown name raises `AttributeError`.

## Passing arguments

IDL `args` map positionally first, then by keyword name:

```python
program.instruction.transfer(1_000, "memo", accounts={...})
program.instruction.transfer(amount=1_000, note="memo", accounts={...})
```

A missing arg raises `ValueError: Argumento obrigatório ausente: <name>`.

`accounts=` is a dict keyed by the IDL account names; every account in the IDL must
be present or you get `ValueError: Conta obrigatória ausente...`. Values may be
`Pubkey` or base58 `str` — both are stringified then re-parsed.

The 8-byte discriminator is computed as `sha256(f"global:{snake_case(name)}")[:8]`,
matching Anchor's convention. It is **not** read from `idl["instructions"][i]["discriminator"]`
even when present.

## Borsh type support

`solinpy.anchor.borsh.serialize(val, type_def, types_registry)` handles:

- Integers: `u8 u16 u32 u64 u128 i8 i16 i32 i64 i128` (u128/i128 packed as two LE u64s,
  i128 via two's complement)
- `f32`, `f64`, `bool`
- `publicKey` — accepts `Pubkey` or base58 `str`
- `string` — u32 length prefix + UTF-8
- `bytes` — hex `str` or any bytes-like; u32 length prefix
- `{"option": T}` — `None` → `\x00`, else `\x01` + payload
- `{"vec": T}` — u32 count + elements
- `{"array": [T, n]}` — fixed, raises `ValueError` on length mismatch
- `{"defined": "Name"}` — resolved against `program.types_registry`, built from
  `idl["types"]`. Structs accept a dict or an object with matching attributes.
  Enums accept a bare `"VariantName"` string for unit variants, or
  `{"VariantName": payload}` for variants with fields.

Anything else raises `ValueError`. There is **no deserializer** — the module is
write-only, so decoding account state means going through
`solinpy.utils.account_decoder` or parsing manually.

## IDL version compatibility — important

The module targets **Anchor ≤ 0.29 IDL format only**. Anchor 0.30+ changed the
schema and will break:

| Field | ≤0.29 (supported) | 0.30+ (unsupported) |
| --- | --- | --- |
| Account flags | `isSigner`, `isMut` | `signer`, `writable` |
| Pubkey type | `"publicKey"` | `"pubkey"` |
| Custom type ref | `{"defined": "Name"}` | `{"defined": {"name": "Name"}}` |

Symptoms: `KeyError: 'isSigner'` when building accounts, or
`ValueError: Tipo básico não suportado: pubkey` during serialization.

If the user has a 0.30+ IDL, the options are: downconvert the IDL, or extend
`program.py:83-84` and `borsh.py:43,76` to accept both shapes. Say which one you're
doing rather than silently patching the IDL.

## Sending is broken in 0.1.7

`_execute_sync` / `_execute_async` pass a `solders.Transaction` to
`client.send_transaction`, which expects a base64 string. The object is not JSON
serializable, so the call fails with an opaque `RPCError`.

Until fixed, build with `program.instruction.*` and send yourself:

```python
import base64
from solders.hash import Hash
from solders.message import Message
from solders.transaction import Transaction

ix = program.instruction.initialize(42, accounts={...})
bh = Hash.from_string(str(client.get_latest_blockhash()))
tx = Transaction([payer], Message([ix], payer.pubkey()), bh)
sig = client.send_transaction(base64.b64encode(bytes(tx)).decode())
```

The fix in-repo is the same one-liner applied at `program.py:131` and `:137`.
