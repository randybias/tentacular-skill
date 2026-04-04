# Enclave Isolation Reference

When The Kraken operates on behalf of users, every message arrives with a
channel context that binds all operations to a specific enclave. Isolation
rules prevent cross-enclave leakage and ensure agents never operate in the
wrong space.

## Channel-Scoped Rule

Every message received in a Slack channel scopes ALL operations to that
channel's enclave. The agent does NOT ask "which enclave?" — the enclave
is resolved from the channel context automatically.

```
Message in #competitor-pricing  →  enclave: competitor-pricing
Message in #infra-alerts         →  enclave: infra-alerts
DM to @Kraken                    →  no enclave scope (cross-enclave context)
```

The Kraken passes `enclave_name` (derived from `channel_id`) to every `tntc`
command and MCP tool call. The agent never hardcodes or guesses an enclave
name.

## Cross-Enclave Operations via DM Only

DMs to The Kraken are the **only** context for cross-enclave operations:

- Listing all enclaves the user has access to
- Comparing tentacle state across enclaves
- Admin tasks that span multiple enclaves
- Any request that requires access to more than one enclave

If a user in a channel asks for something that requires cross-enclave access
(e.g., "Show me all my tentacles across all channels"), The Kraken declines
in the channel and invites the user to DM instead:

> "Cross-enclave operations can only be run via DM. Message me directly."

## Enclave Context Resolution

The Kraken resolves `enclave_name` from `channel_id` at the start of every
request:

1. Look up the enclave registered to this `channel_id` via `enclave_info`
2. If no enclave is registered: prompt the user to provision one or clarify intent
3. Pass `enclave_name` to all downstream tool calls — never infer it from user text

Tool calls use the resolved name, not the channel display name:

```
# CORRECT — resolved from channel_id
enclave_info(name="competitor-pricing")
tntc deploy price-monitor --enclave competitor-pricing

# WRONG — inferred from message text
enclave_info(name="Competitor Pricing")
```

Channel renames do NOT change the enclave name slug. Always use the slug
from `enclave_info`, not the current Slack channel display name.

## Tentacle Path Scoping

On disk, tentacles live under `~/tentacles/<enclave-name>/<tentacle-name>/`.

```
~/tentacles/
  competitor-pricing/    ← enclave directory
    price-monitor/       ← tentacle
    alert-dispatcher/    ← tentacle
  infra-alerts/          ← another enclave
    node-health/         ← tentacle
```

Rules:
- The agent MUST create tentacles inside `~/tentacles/<enclave-name>/`, never at `~/tentacles/<name>/` (flat, pre-enclave layout) and never outside the enclave directory.
- Use `tntc init <tentacle-name>` from inside the enclave directory, or `tntc scaffold init <scaffold> <tentacle-name> --enclave <enclave-name>`.
- Never create a tentacle in a different enclave's directory, even if the names
  are similar.

## Three-Layer Consistency

Every tentacle that exists in one layer should exist in all three:

| Layer | Location | Authority |
|-------|----------|-----------|
| **Git** | `active/<enclave>/<tentacle>/` in the state repo | System of record (when git-state is enabled) |
| **Disk** | `~/tentacles/<enclave>/<tentacle>/` | Working copy |
| **K8s** | Namespace `<enclave>`, Deployment `<tentacle>` | Deployed runtime |

Before deploying, verify that the tentacle path is under the correct enclave
directory. If disk and K8s layers disagree on enclave, investigate before
proceeding.

## Ambiguous Context

When context is unclear, the agent asks rather than guessing.

**Ambiguous tentacle name in DM:**

> User: "Can you update the price-monitor?"
> (DM context — no enclave resolved)

The agent asks: "Which enclave is `price-monitor` in? I found it in both
`competitor-pricing` and `market-research`. Please specify."

**User references wrong enclave in a channel:**

> User in #infra-alerts: "Deploy the price-monitor"

`price-monitor` does not exist in `infra-alerts`. The agent clarifies:

> "I don't see a `price-monitor` in the `infra-alerts` enclave. Did you mean
> to ask in #competitor-pricing, or would you like to create a new
> `price-monitor` here?"

**Tentacle exists in multiple enclaves (DM only):**

Always ask which enclave before taking any action. Never default to the
alphabetically first match or the most recently deployed one.

## Common Mistakes

| Mistake | What Happens | Fix |
|---------|--------------|-----|
| Operating on the wrong enclave because user mentioned it by display name | Wrong tentacles modified | Always resolve `enclave_name` via `channel_id`, not user text |
| Creating tentacle at `~/tentacles/<name>/` (flat path) | Tentacle outside enclave scope | Create at `~/tentacles/<enclave-name>/<name>/` |
| Performing cross-enclave queries from a channel context | Leaks cross-enclave information | Decline and redirect to DM |
| Inferring enclave from channel display name after a rename | Name mismatch, tool call fails | Use slug from `enclave_info`, not display name |
| Assuming a tentacle name is unique across the platform | Wrong tentacle modified in DM context | Always ask for enclave when context is a DM |
