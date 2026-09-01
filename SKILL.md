---
name: solinpy-sdk
description: Build Solana integrations with the SolInPy Python SDK — RPC calls (sync/async), wallet creation and import (JSON keypair, BIP39 mnemonic), devnet/testnet airdrops, SPL token transfers and mints, account data decoding, and dynamic Anchor program calls from an IDL. Use whenever the task involves `solinpy`, `SolanaRPCClient`, `WalletManager`, `send_token_transfer`, `request_airdrop`, `solinpy.anchor.Program`, or when writing Python code against Solana inside a repo that depends on solinpy.
---

# SolInPy SDK

Python SDK for Solana. Thin, dependency-light layer over `solders` that handles
JSON-RPC transport (retries, backoff, friendly errors), wallets, SPL tokens, and
Anchor IDL calls.

Target version: **0.1.7**. Requires Python 3.10+.

## Before writing code

1. Confirm the installed version — the API is pre-1.0 and moves:
   ```bash
   python -c "import solinpy, importlib.metadata as m; print(m.version('solinpy'))"
   ```
2. **Default to `devnet`.** Only use `mainnet` when the user explicitly asks. Real
   funds move irreversibly and there is no confirmation prompt inside the SDK.
3. There are **no top-level re-exports** — `solinpy/__init__.py` is empty. Always
   import from the full path (`from solinpy.client.client import SolanaRPCClient`),
   never `from solinpy import SolanaRPCClient`.

## Core flow

```python
from solinpy.client.client import SolanaRPCClient
from solinpy.client.entities import RPCConfig
from solinpy.wallet.manager import WalletManager
from solinpy.utils.airdrop import request_airdrop

client = SolanaRPCClient(RPCConfig(cluster="devnet"))   # or SolanaRPCClient("https://...")
wallet = WalletManager.generate_keypair()

request_airdrop(wallet, cluster="devnet")               # blocks until confirmed
print(client.get_sol_balance(str(wallet.pubkey())))
```

`RPCConfig(cluster=...)` accepts `"devnet"` (default), `"testnet"`, `"mainnet"`.
Unknown values silently fall back to devnet. Pass `custom_endpoint=` for a private
RPC — it wins over `cluster`.

## Module map

| Need | Import from |
| --- | --- |
| Sync RPC | `solinpy.client.client.SolanaRPCClient` |
| Async RPC (httpx) | `solinpy.client.async_client.SolanaAsyncRPCClient` |
| Config / errors | `solinpy.client.entities.RPCConfig`, `solinpy.client.execptions.RPCError` |
| Wallets | `solinpy.wallet.manager.WalletManager`, `solinpy.wallet.mnemonic.import_from_mnemonic` |
| Airdrop | `solinpy.utils.airdrop.request_airdrop` / `create_airdrop` |
| SPL tokens | `solinpy.transaction.token` |
| Account decoding | `solinpy.utils.account_decoder` |
| Anchor programs | `solinpy.anchor.program.Program` |

Note the module is spelled `execptions` (typo is in the shipped package — importing
`exceptions` fails).

Full signatures and return shapes: `references/api-reference.md`.

## Known defects — read before shipping code

These are live bugs in 0.1.7. Working around them is usually correct; fixing them
in-repo is better when the task is SDK maintenance.

**`send_token_transfer` passes a `Transaction` object to a method that expects
base64.** `client.send_transaction(tx_base64: str)` puts its argument straight into
the JSON-RPC payload, so a `solders.Transaction` raises `TypeError`, which the
generic handler rewraps as an opaque `RPCError("Falha inesperada...")`. Same defect
in `anchor.program` `_execute_sync` / `_execute_async`. When sending manually:

```python
import base64
sig = client.send_transaction(base64.b64encode(bytes(tx)).decode())
```

**`get_token_accounts_by_owner` crashes on `Pubkey`** despite its `str | Pubkey`
annotation — it calls `.strip()` directly. Always pass `str(pubkey)`.

**ATA creation is not idempotent.** `create_associated_token_account` emits
instruction data `b""` (Create). Since existence is checked in a separate RPC call
before sending, a race makes the whole transaction fail. Use `b"\x01"`
(CreateIdempotent) when building the instruction by hand.

**Anchor support is IDL ≤0.29 only.** See `references/anchor.md`.

Full list with file:line: `references/pitfalls.md`.

## Wallets

```python
kp = WalletManager.generate_keypair()
kp = WalletManager.import_from_json("my-wallet.json")        # Solana CLI format
from solinpy.wallet.mnemonic import import_from_mnemonic
kp = import_from_mnemonic("word word ...")                    # BIP39, 12/18/24 words
```

`import_from_mnemonic` derives via SLIP-0010 Ed25519, default path `m/44'/501'/0'/0'`.
Only hardened indices are valid — a path segment without `'` raises `ValueError`.
This path may not match Phantom/Solflare defaults; if an imported address doesn't
match what the user expects, try `m/44'/501'/0'` or `m/44'/501'/0'/0'` explicitly.

**Never write a generated keypair to a file inside the repo, and never print a
secret key.** If a task requires persisting one, put it outside the working tree or
add it to `.gitignore` first, and say so. Print `kp.pubkey()`, not `bytes(kp)`.

## SPL tokens

```python
from solinpy.transaction.token import send_token_transfer

sig = send_token_transfer(
    client=client,
    sender_keypair=wallet,
    destination_wallet="<base58>",
    token_mint="<mint>",
    amount=25_000_000,     # raw base units, NOT UI amount
    decimals=6,
)
```

`amount` is in base units — the caller multiplies by `10**decimals`. `decimals` must
match the mint exactly or the on-chain `transfer_checked` fails. Read it from the
mint rather than hardcoding.

Lower-level instruction builders (`initialize_mint`, `mint_to`, `set_authority`,
`transfer_checked`, `get_associated_token_address`) are in the same module and
return `solders.Instruction` — compose them into a `Message`/`Transaction` yourself.
See `references/api-reference.md` and the repo's `examples/mint_nft.py`.

## Error handling

Every RPC failure surfaces as `RPCError` with `.message`, `.code`, `.method`,
`.context`, `.cause`. Messages are already translated to actionable Portuguese text
(insufficient funds, expired blockhash, unknown account). Catch that one type:

```python
from solinpy.client.execptions import RPCError
try:
    client.get_balance(addr)
except RPCError as e:
    print(e.code, e.method, e.context)
```

Transient failures (HTTP 429/5xx, RPC `-32004`/`-32005`, connection errors) are
retried automatically with exponential backoff + jitter: `max_retries=3`,
`base_delay=1.0`, `max_delay=30.0`, all tunable on `RPCConfig`.

## Testing

Both clients take an injected transport — use it instead of monkeypatching:

```python
SolanaRPCClient(cfg, transport=fake_urlopen)      # sync: urlopen-compatible callable
SolanaAsyncRPCClient(cfg, client=httpx.AsyncClient(transport=...))
```

The repo has a mock at `solinpy/client/rpc_mock.py`. Tests hitting the real network
must be marked `@pytest.mark.integration`; the default suite runs
`pytest -m "not integration"`.

## When working *on* the SDK repo

CI runs `ruff check .`, `mypy solinpy` (strict), `pytest -q`. Run all three before
claiming done. Line length is 100. `mypy` strict means every new function needs
annotations; test modules are excluded via `pyproject.toml` overrides.

## References

- `references/api-reference.md` — every public signature, return shape, and the
  async client's parity gaps.
- `references/anchor.md` — loading an IDL, calling instructions, Borsh type support,
  and the 0.30+ incompatibility.
- `references/pitfalls.md` — full defect list with file:line and workarounds.
