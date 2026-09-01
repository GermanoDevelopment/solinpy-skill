# solinpy-sdk skill

A [Claude Code](https://claude.com/claude-code) skill for the
[SolInPy](https://pypi.org/project/solinpy/) Python SDK for Solana.

It gives an agent the SDK's real API surface — correct import paths, return shapes,
argument units — plus the guardrails that matter when code can move funds on a live
network.

## Install

```bash
git clone git@github.com:GermanoDevelopment/solinpy-skill.git ~/.claude/skills/solinpy-sdk
```

The directory name matters: Claude Code resolves the skill by folder, so clone it as
`solinpy-sdk`.

Verify it loaded by asking Claude something like *"use solinpy to check a devnet
wallet balance"* — the skill should trigger on its own.

## Layout

```
SKILL.md                       # always loaded: core flow, module map, guardrails
references/
├── api-reference.md           # every public signature and return shape
├── anchor.md                  # IDL loading, Borsh types, 0.30+ compatibility
└── pitfalls.md                # known defects with workarounds
```

`SKILL.md` stays small on purpose. The reference files are read on demand, only when
the task actually reaches Anchor programs or an obscure return shape.

## What it covers

- **RPC** — `SolanaRPCClient` and the async `SolanaAsyncRPCClient`, retry/backoff
  config, and `RPCError` handling.
- **Wallets** — keypair generation, Solana CLI JSON import, BIP39 mnemonic import
  with SLIP-0010 Ed25519 derivation.
- **Airdrops** — devnet/testnet, with the blocking confirmation loop.
- **SPL tokens** — transfers, mints, ATA derivation, and the instruction builders.
- **Account decoding** — token accounts, mints, and the auto-detection heuristic's
  limits.
- **Anchor** — calling programs dynamically from an IDL.

## Why the defects are documented

`references/pitfalls.md` lists real bugs in the SDK, not hypotheticals. An agent that
calls `send_token_transfer` without knowing the transaction never serializes will ship
code that fails with an opaque error. Each entry carries both a workaround for the
call site and the fix for the SDK repo, so the agent can pick based on the task.

## Version

Written against **SolInPy 0.1.7**. The SDK is pre-1.0 and its API moves — when a
documented defect gets fixed upstream, update `pitfalls.md` in the same pass or the
skill starts lying.

## Related

- SDK source: [carcaras/solinpy](https://github.com/carcaras/solinpy)
- Skill authoring format: [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills)
