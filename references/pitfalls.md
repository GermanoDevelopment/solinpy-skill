# SolInPy 0.1.7 — known defects and rough edges

Verified by reading the source at commit `3559a5e`. Line references are into the
`solinpy` package. Re-check against the installed version before relying on any of
these — they are expected to be fixed over time.

## Blocking bugs

### 1. `send_transaction` receives an object, expects base64

`client/client.py:173` and `client/async_client.py:161` declare
`send_transaction(tx_base64: str, ...)` and drop the value into the JSON-RPC params.
`_json_safe` only converts `Pubkey` and objects with a `.pubkey()` method, so a
`solders.Transaction` reaches `json.dumps` and raises `TypeError` — caught by the
broad `except Exception` in `_call` and rewrapped as
`RPCError("Falha inesperada ao executar sendTransaction.")`.

Callers hitting this: `transaction/token.py:233` (`send_token_transfer`),
`anchor/program.py:131` and `:137`.

Workaround at the call site:

```python
import base64
sig = client.send_transaction(base64.b64encode(bytes(tx)).decode())
```

In-repo fix: coerce inside `send_transaction` —

```python
if not isinstance(tx_base64, str):
    tx_base64 = base64.b64encode(bytes(tx_base64)).decode()
```

### 2. `get_token_accounts_by_owner` rejects `Pubkey`

`client/client.py:195` annotates `address: str | Pubkey` but calls `address.strip()`
→ `AttributeError` on `Pubkey`. `_normalize_address` exists three functions above and
is not used. `get_transaction_history` (`client.py:256`) has the same raw `.strip()`
but is annotated `str`, so it is only a latent issue.

Workaround: pass `str(pubkey)`. The async client does not have this bug.

### 3. ATA creation is not idempotent

`transaction/token.py:41` sets `data=b""` — the Associated Token Account program's
`Create` instruction, which fails if the account already exists.
`send_token_transfer` checks existence in a separate RPC round-trip first, so a
concurrent creation between check and send aborts the entire transaction, including
the transfer.

Fix: `data=b"\x01"` (`CreateIdempotent`). Safe to apply unconditionally.

## Packaging

### 4. No `[build-system]` in `pyproject.toml`

No build backend and no `[tool.setuptools.packages]` / `[tool.hatch.build]` section.
`pip install .` relies on setuptools auto-discovery, which packages `solinpy/tests/`,
`solinpy/client/test_client.py`, `test_async_client.py`, `rpc_mock.py`, and
`conftest.py` into the published wheel.

### 5. `requirements.txt` conflicts with `pyproject.toml`

`requirements.txt` pins `solders==0.27.1` / `solana==0.36.11` exactly while
`pyproject.toml` declares `>=`. It also mixes dev tooling (pytest, mypy, ruff) into
runtime requirements. Dev deps belong in `[project.optional-dependencies]`.

### 6. No top-level exports

`solinpy/__init__.py` and `solinpy/transaction/__init__.py` are empty (0 bytes).
Only `solinpy.client` and `solinpy.utils` define `__all__`. Every import must use
the full dotted path.

## Naming and API friction

- **`execptions.py`** — the module name is misspelled and it is part of the public
  import path. `from solinpy.client.exceptions import RPCError` fails.
- **`BlockhashResult(str)`** (`client.py:30`) — a `str` subclass whose `.value`
  returns itself. Exists to keep three call shapes working at once; it forces the
  four-branch type sniffing in `anchor/program.py:116-123`. Just use `str(bh)`.
- **`get_account_info` returns a locally-defined class** (`client.py:167`) — not
  importable, not type-checkable. Only `.value` is usable.
- **`RPCConfig` duplicate fields** — `retries`/`max_retries` and
  `backoff_factor`/`base_delay` are reconciled in `__post_init__`. Set one per pair.
- **Portuguese error messages, English docstrings** — `execptions.py`, `borsh.py`,
  `program.py`, and `airdrop.py` raise pt-BR text while the README and
  `wallet/manager.py` are in English. Don't match either blindly; follow whatever the
  surrounding file already does.

## Security notes

- Never commit a keypair file. The Solana CLI JSON format that
  `WalletManager.import_from_json` reads is a plain 64-byte secret key — anyone with
  the file controls the wallet. Keep wallet files outside the working tree, or add
  them to `.gitignore` before generating one. If a keypair ever lands in git history,
  it is compromised permanently: rotate it, don't just delete the file.
- `wallet/mnemonic.py:28-29` does `del seed, private_key; gc.collect()` as a
  scrubbing gesture. Python `bytes` are immutable — this does not zero memory. Don't
  present it to a user as a security guarantee.

## Async client performance

`async_client.py:73` builds a fresh `httpx.AsyncClient()` on every `_call` when none
was injected, discarding connection reuse and paying a TLS handshake per request.
Inject a shared client for any loop or batch.
