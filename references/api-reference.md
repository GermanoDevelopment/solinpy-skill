# SolInPy 0.1.7 — API reference

Every path below is the real import path. There are no top-level re-exports.

## `solinpy.client.entities`

```python
@dataclass
class RPCConfig:
    cluster: str = "devnet"                    # devnet | testnet | mainnet; unknown → devnet
    custom_endpoint: Optional[str] = None      # overrides cluster when set
    timeout: float = 10.0
    max_retries: int = 3                       # total attempts = max_retries + 1
    base_delay: float = 1.0
    max_delay: float = 30.0
    retryable_http_codes: tuple[int, ...] = (429, 500, 502, 503, 504)
    retryable_rpc_codes: tuple[int, ...] = (-32004, -32005)
    retries: Optional[int] = None              # legacy alias → max_retries
    backoff_factor: Optional[float] = None     # legacy alias → base_delay
```

`__post_init__` reconciles the alias pairs in both directions. Set one of each pair,
not both.

`TokenBalance(mint, amount, decimals, ui_amount)` is declared but not returned by
any client method — `get_token_balances` returns plain dicts.

## `solinpy.client.client.SolanaRPCClient`

```python
SolanaRPCClient(config: RPCConfig | str | None = None,
                transport: Callable[..., Any] | None = None)
```

A `str` config is treated as `custom_endpoint`. `transport` defaults to
`urllib.request.urlopen` and must be a context-manager-returning callable with the
same `(request, timeout=...)` shape.

| Method | Returns |
| --- | --- |
| `get_health()` | `str` — `"ok"` |
| `get_latest_blockhash(commitment="confirmed")` | `BlockhashResult` (a `str` subclass) |
| `get_account_info(address, commitment="confirmed")` | object with `.value` (`None` if the account does not exist) |
| `send_transaction(tx_base64: str, max_retries=5)` | `str` signature |
| `get_balance(address)` | `int` lamports |
| `get_sol_balance(address)` | `float` SOL |
| `get_token_accounts_by_owner(address)` | `list[dict]` raw `jsonParsed` accounts |
| `get_token_balances(address)` | `list[{"mint", "amount", "decimals"}]` — `amount` is `uiAmount` |
| `get_transaction_history(address, limit=20, before=None, until=None, commitment="confirmed")` | `list[dict]` signature records |

`BlockhashResult` subclasses `str` and exposes `.value` (returns itself) and
`.blockhash` (returns `str(self)`), so `bh`, `bh.value.blockhash`, and `bh.blockhash`
are all the same string. Prefer plain `str(bh)`.

`get_account_info` returns an instance of a class defined *inside* the method —
don't try to import or isinstance-check it. Only `.value` is meaningful.

`_call(method, params, context)` is private but is the escape hatch for RPC methods
the SDK doesn't wrap. It returns the full response body (`{"jsonrpc", "id", "result"}`),
so index `["result"]` yourself. `solinpy.utils.airdrop` uses it this way.

## `solinpy.client.async_client.SolanaAsyncRPCClient`

```python
SolanaAsyncRPCClient(config: RPCConfig | str | None = None,
                     client: httpx.AsyncClient | None = None)
```

Same method names, all `async`. Two differences from the sync client:

- `get_balance`, `get_sol_balance`, `get_token_balances`, `get_token_accounts_by_owner`
  correctly accept `str | Pubkey` here (the sync `get_token_accounts_by_owner` does not).
- When no `client` is injected it constructs a fresh `httpx.AsyncClient()` **per call**
  — no connection pooling. For anything beyond a couple of requests, inject one:
  ```python
  async with httpx.AsyncClient() as h:
      c = SolanaAsyncRPCClient(RPCConfig(cluster="devnet"), client=h)
  ```

## `solinpy.client.execptions.RPCError`

```python
RPCError(message, *, code=None, method=None, context=None, cause=None)
```

Attributes: `.message`, `.code`, `.method`, `.context: dict`, `.cause`.
`str(e)` renders `message [código RPC=… | método=… | contexto=… | causa=…]`.

Constructors used internally: `RPCError.from_rpc_error(method, rpc_error, context=)`
and `RPCError.from_transport_error(method, error, context=, message=)`.

Friendly-message mapping covers: `-32600` invalid request, `-32601` unknown method,
`-32602` invalid params, insufficient funds/balance, account not found, blockhash
not found/expired, `-32004`/`-32005` node unavailable. Everything else falls through
to `Erro RPC ao executar {method}: {raw}`. All messages are Portuguese.

## `solinpy.wallet`

```python
WalletManager.generate_keypair() -> Keypair
WalletManager.import_from_json(file_path: str | Path) -> Keypair
    # raises FileNotFoundError, or ValueError if the JSON isn't a list of ints

import_from_mnemonic(mnemonic_str: str,
                     passphrase: str = "",
                     derivation_path: str = "m/44'/501'/0'/0'") -> Keypair
    # ValueError on: non-str input, word count not in (12, 18, 24),
    #                BIP39 checksum failure, path not starting with "m",
    #                any non-hardened segment (missing trailing ')
```

All returns are `solders.keypair.Keypair`.

## `solinpy.utils.airdrop`

```python
create_airdrop(keypair, cluster="devnet", lamports=1_000_000_000,
               timeout=60.0, poll_interval=2.0, custom_endpoint=None) -> dict
request_airdrop(...)   # identical signature, backward-compatible alias
```

Returns `{"signature": str, "confirmed": True, "balance": int}`. Blocks, polling
`getSignatureStatuses` every `poll_interval` until `confirmed`/`finalized`, then
raises `TimeoutError` at `timeout`. Raises `ValueError` for any cluster other than
`devnet`/`testnet`. Builds its own client internally — the caller's client is not reused.

## `solinpy.transaction.token`

Constants: `TOKEN_PROGRAM_ID`, `ASSOCIATED_TOKEN_PROGRAM_ID` (both `Pubkey`).
`AuthorityType.MintTokens|FreezeAccount|AccountOwner|CloseAccount` = 0|1|2|3.

```python
get_associated_token_address(owner_address: str, token_mint_address: str) -> Pubkey
create_associated_token_account(payer: Pubkey, owner: Pubkey, mint: Pubkey) -> Instruction
transfer_checked(TransferCheckedParams(program_id, source, mint, dest, owner, amount, decimals)) -> Instruction
initialize_mint(InitializeMintParams(program_id, mint, decimals, mint_authority, freeze_authority=None)) -> Instruction
mint_to(MintToParams(program_id, mint, dest, mint_authority, amount)) -> Instruction
set_authority(SetAuthorityParams(program_id, account, authority_type, current_authority, new_authority=None)) -> Instruction

send_token_transfer(client, sender_keypair, destination_wallet, token_mint,
                    amount: int, decimals: int) -> Any
```

Both address args of `get_associated_token_address` are **strings**, not `Pubkey` —
wrap with `str()`.

`initialize_mint` emits opcode 20 (`InitializeMint2`), so it takes only the mint
account — no rent sysvar. Create and fund the mint account with
`solders.system_program.create_account` in the same transaction, 82 bytes, owner
`TOKEN_PROGRAM_ID`.

`send_token_transfer` derives both ATAs, prepends a create-ATA instruction when
`client.get_account_info(receiver_ata).value is None`, then builds and sends. See
`pitfalls.md` — the send step is broken in 0.1.7.

## `solinpy.utils.account_decoder`

```python
decode_base64_to_bytes(data: str) -> bytes          # ValueError on bad base64
decode_bytes_to_base64(data: bytes) -> str
parse_pubkey_from_bytes(data, offset=0) -> Pubkey
parse_u64_from_bytes(data, offset=0, little_endian=True) -> int
parse_u32_from_bytes(data, offset=0, little_endian=True) -> int
parse_u8_from_bytes(data, offset=0) -> int
decode_system_account(data: bytes) -> dict
decode_spl_token_account(data: bytes) -> dict
decode_spl_token_mint(data: bytes) -> dict
decode_account_data(data: bytes | str | dict, program_type="auto") -> dict
```

`decode_account_data` dispatch:

- `dict` in → `{"program": "jsonParsed", "parsed_data": ...}` (passthrough).
- `program_type="auto"` guesses by length: exactly 82 → mint, ≥72 → token account,
  else raw. **This is a heuristic** — any 82-byte account is read as a mint. Pass
  `program_type` explicitly (`"spl_token"`, `"spl_token_mint"`, `"system"`, `"raw"`)
  whenever the owner program is known.
- Decode failures fall back to `{"program": "raw", "data_hex", "data_length", "encoding"}`
  instead of raising.

Token account result keys: `program`, `mint`, `owner`, `amount`, `delegate`, `state`
(`uninitialized`/`initialized`/`frozen`), `is_native`, `native_amount`,
`delegated_amount`, `close_authority`.

Mint result keys: `program`, `mint_authority`, `supply`, `decimals`, `is_initialized`,
`freeze_authority`.

Feed it the base64 payload from `getAccountInfo`:

```python
info = client.get_account_info(addr)
decoded = decode_account_data(info.value["data"][0], program_type="spl_token_mint")
```
