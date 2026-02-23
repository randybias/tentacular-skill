# Phase 01: Install tntc

**Gate: `tntc version` must succeed before any other work.**

## Check

```bash
which tntc && tntc version
```

If this succeeds, you are done with this phase. Proceed to `phases/02-configure.md`.

## Install

If `tntc` is missing:

```bash
curl -fsSL https://raw.githubusercontent.com/randybias/tentacular/main/install.sh | sh
```

Then verify:

```bash
tntc version
```

## Verify

| Output | Meaning | Action |
|--------|---------|--------|
| `tntc v1.x.x` or similar | Installed correctly | Proceed to phase 02 |
| `dev (commit none, built unknown)` | Built from source — all commands functional | Proceed to phase 02 |
| Command not found / error | Install failed | Stop. Surface error to user. Do not proceed. |

**If `tntc version` fails: stop completely.** Do not attempt any other tntc commands.
Surface the exact error and ask the user to resolve it before continuing.

## Gate Verification

Before leaving this phase, confirm:

- [ ] `tntc version` exits 0
- [ ] Output shows a version string (release tag or `dev`)

Only then: proceed to `phases/02-configure.md`.
