# Secrets Reference

How tentacle secrets work end-to-end: the `$shared` reference format, workspace
shared-secret files, engine resolution, and the correct pattern for calling
external APIs (including Slack) from workflow nodes.

**Read this before writing any node that calls an external API.** Every
`.secrets.yaml.example` in the scaffolds repo teaches a non-functional format.
The correct mechanism is documented here.

---

## Architecture: Three Layers

Three components handle secrets in sequence. Understanding all three is required
to provision secrets correctly.

### Layer 1 — tntc deploy (`pkg/cli/deploy.go`)

`tntc deploy` reads `.secrets.yaml` from the tentacle directory and calls
`buildSecretFromYAML`. It enforces two rules:

1. The file MUST be a **flat map** of `<group>: $shared.<name>`. No `secrets:`
   wrapper key. No nested structure.
2. **Every value MUST be a `$shared.<name>` reference.** Direct values (e.g.,
   `openai: sk-proj-abc`) are rejected with: `all secrets must use $shared.<name>
   references`. Deployment fails immediately.

### Layer 2 — $shared resolution (`pkg/cli/secrets.go`)

`resolveSharedSecrets` resolves each `$shared.<name>` reference:

1. Reads the file at `<workspace>/.secrets/<name>`.
   The workspace root comes from the `workspace:` key in
   `~/.tentacular/config.yaml` (typically `~/tentacles`).
2. Attempts to JSON-parse the file content. If it parses as JSON, the full
   parsed object is the resolved value. If it does not parse, the raw string
   is the value.
3. Builds a Kubernetes Secret with one key per top-level `.secrets.yaml` entry.
   The key name is the entry key (e.g., `openai`), and the value is the resolved
   object (JSON-serialized if it was an object).

### Layer 3 — Engine resolution (`engine/context/secrets.ts`)

The k8s Secret is mounted at `/app/secrets/` inside the workflow pod. The engine
calls `loadSecretsFromDir` at startup:

- Each FILE in `/app/secrets/` becomes a top-level entry in the in-memory
  secrets store.
- If the file content is valid JSON, the entry is the parsed object (nested).
- If the content is a plain string, the entry is `{ value: "<string>" }`.

`ctx.dependency(name).secret` reads the contract's `auth.secret` string and
**splits it on `.`** to traverse the nested secrets object:

```
auth.secret: "openai.api_key"
  → secrets["openai"]["api_key"]
```

This means the file at `~/tentacles/.secrets/openai` MUST contain JSON keyed by
the subkey referenced in the contract. A flat dotted key does not work —
see Common Mistakes below.

---

## Correct Setup: Complete Worked Example

The following example provisions secrets for a tentacle that calls OpenAI and
posts results to Slack.

### Step 1: Contract (`workflow.yaml`)

Declare every external dependency with its auth path as `group.subkey`:

```yaml
contract:
  dependencies:
    openai-api:
      protocol: https
      host: api.openai.com
      port: 443
      auth:
        type: api-token
        secret: openai.api_key      # group=openai, subkey=api_key
    slack:
      protocol: https
      host: slack.com
      port: 443
      auth:
        type: api-token
        secret: slack.bot_token     # group=slack, subkey=bot_token
```

### Step 2: Per-tentacle `.secrets.yaml`

This file lives in the tentacle directory alongside `workflow.yaml`. It is
git-ignored. It contains only `$shared.<name>` references — never literal
values:

```yaml
openai: $shared.openai
slack: $shared.slack
```

The key on the left (`openai`, `slack`) is the group name. It MUST match the
first segment of the `auth.secret` path in the contract.

### Step 3: Workspace-root shared files

Create one file per group in `~/tentacles/.secrets/`. This directory is
git-ignored globally. Files MUST be valid JSON keyed by the subkeys referenced
in the contract:

```
~/tentacles/.secrets/openai
```
```json
{"api_key": "sk-proj-..."}
```

```
~/tentacles/.secrets/slack
```
```json
{"bot_token": "xoxb-...", "webhook_url": "https://hooks.slack.com/..."}
```

```
~/tentacles/.secrets/anthropic
```
```json
{"api_key": "sk-ant-..."}
```

File permissions should be 0600:

```bash
chmod 600 ~/tentacles/.secrets/*
```

### Step 4: How it resolves

`tntc deploy` reads `.secrets.yaml`, resolves `$shared.openai` to the content of
`~/tentacles/.secrets/openai`, and creates a Kubernetes Secret
`<tentacle-name>-secrets` with:

| k8s Secret key | Value |
|----------------|-------|
| `openai` | `{"api_key":"sk-proj-..."}` |
| `slack` | `{"bot_token":"xoxb-...","webhook_url":"..."}` |

The engine mounts this Secret and loads it. `ctx.dependency("openai-api").secret`
traverses `secrets["openai"]["api_key"]` and returns `"sk-proj-..."`.

### Step 5: Node code

`ctx.dependency(name).secret` returns the resolved string value. Use it to set
auth headers:

```typescript
import type { Context } from "tentacular";

export default async function run(ctx: Context, input: unknown): Promise<unknown> {
  const openai = ctx.dependency("openai-api");
  if (!openai.secret) {
    ctx.log.error("openai.api_key is empty — check ~/tentacles/.secrets/openai");
    return { error: "missing secret" };
  }

  const resp = await globalThis.fetch("https://api.openai.com/v1/chat/completions", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${openai.secret}`,
    },
    body: JSON.stringify({ model: "gpt-4o-mini", messages: [{ role: "user", content: "Hello" }] }),
  });
  const data = await resp.json();
  return { reply: data.choices[0].message.content };
}
```

---

## Calling Slack from a Tentacle

Use `ctx.dependency("slack")` and call the Slack Web API directly. This is the
only correct pattern for Slack integration in workflow nodes.

```typescript
import type { Context } from "tentacular";

export default async function run(ctx: Context, input: unknown): Promise<unknown> {
  const slack = ctx.dependency("slack");
  if (!slack.secret) {
    ctx.log.error("slack.bot_token is empty — check ~/tentacles/.secrets/slack");
    return { error: "missing slack secret" };
  }

  const payload = input as { channel: string; text: string };
  const resp = await globalThis.fetch("https://slack.com/api/chat.postMessage", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${slack.secret}`,
    },
    body: JSON.stringify({ channel: payload.channel, text: payload.text }),
  });

  const result = await resp.json() as { ok: boolean; error?: string };
  if (!result.ok) {
    ctx.log.error(`Slack API error: ${result.error}`);
    return { error: result.error };
  }
  return { posted: true, channel: payload.channel };
}
```

**NEVER write to `outbound.ndjson` from workflow node code.** That path is
Kraken-internal and is NOT mounted in the workflow pod. Writing to it silently
does nothing. Slack messages from a tentacle go through `ctx.dependency("slack")`
and `chat.postMessage`, not through any Kraken IPC mechanism.

---

## Secret Provisioning Workflow

When setting up secrets for a new tentacle:

```bash
# 1. Verify tntc secrets init creates a correct placeholder
cd ~/tentacles/<enclave>/<tentacle>
tntc secrets init
# Review the generated .secrets.yaml — confirm it uses $shared.<name> references

# 2. Create the workspace-root shared files if they do not exist
mkdir -p ~/tentacles/.secrets
# For each group:
echo '{"api_key":"sk-proj-..."}' > ~/tentacles/.secrets/openai
chmod 600 ~/tentacles/.secrets/openai

# 3. Verify secrets check passes before deploying
tntc secrets check
```

`tntc secrets check` validates that:
- All `$shared.<name>` references in `.secrets.yaml` resolve to an existing file
  in `~/tentacles/.secrets/`
- No direct values are present (would be caught by deploy anyway)

Run this as Gate 2 in the testing sequence before any deploy.

---

## Two Different `.secrets/` Directories

This is the most common source of confusion. There are two separate locations,
with different roles:

| Location | Role | Committed? |
|----------|------|------------|
| `~/tentacles/<enclave>/<tentacle>/.secrets/` | Per-tentacle placeholder (may be empty or contain `.secrets.yaml.example`) | Structure committed; actual values git-ignored |
| `~/tentacles/.secrets/` | Workspace-root real values, one file per group | Never committed; 0600 permissions |

The per-tentacle `.secrets.yaml` (not `.secrets/`) is the reference map. The
workspace-root `.secrets/<name>` files are the actual values. `tntc deploy`
reads both.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---------|--------------|-----|
| Flat dotted key in shared file: `openai.api_key: "sk-..."` | Engine mounts as `secrets["openai.api_key"]`; `secrets["openai"]["api_key"]` is `undefined`; node gets empty secret; silently succeeds | JSON file with nested key: `{"api_key": "sk-..."}` in `~/tentacles/.secrets/openai` |
| Direct value in `.secrets.yaml`: `openai: sk-proj-abc` | `tntc deploy` rejects immediately: "all secrets must use `$shared.<name>` references" | Use `openai: $shared.openai` |
| `secrets:` wrapper in `.secrets.yaml` | Schema validation fails; deploy rejected | File must be a flat map, no wrapper key |
| Wrong group name: `.secrets.yaml` key does not match contract `auth.secret` first segment | `ctx.dependency().secret` returns empty (group not found in secrets store) | Align `.secrets.yaml` key with contract group. E.g., contract `auth.secret: openai.api_key` → `.secrets.yaml` key `openai` → file `~/tentacles/.secrets/openai` |
| Writing to `outbound.ndjson` to post to Slack | File is not mounted in workflow pod; write silently fails or writes to nowhere | Use `ctx.dependency("slack")` + `chat.postMessage` |
| Hardcoding credentials in node source code | Credentials committed to git; credential gate (`tntc validate`) blocks deploy | Use `ctx.dependency(name).secret`; provision via `~/tentacles/.secrets/` |
| `.gitignore` inline comment: `*.dec.yaml  # decrypted` | Git treats the whole string including `# ...` as the pattern, so `*.dec.yaml` files are NOT ignored | Comments must be on their own line in `.gitignore` |
| Assuming empty secret means deploy failed | Empty secrets produce `success: true` from deploy; failure is silent at runtime when node early-returns | Always run `tntc secrets check` before deploy; check `dep.secret` in node code and log/return an error if empty |

---

## Secret Naming Conventions

Use lowercase, hyphen-free group names that match the external service. The
group name appears in the `.secrets.yaml` file, in the workspace-root `.secrets/`
filename, and in the contract `auth.secret` first segment — they must all match.

Established group names in use:

| Group | Subkeys | Service |
|-------|---------|---------|
| `openai` | `api_key` | OpenAI API |
| `anthropic` | `api_key` | Anthropic API |
| `slack` | `bot_token`, `webhook_url` | Slack Web API |
| `azure` | `sas_token` | Azure Blob / SAS |
| `github` | `token` | GitHub API |

These are conventions, not schema. Use any group/subkey names you choose as long
as they are consistent across the contract, `.secrets.yaml`, and the shared file.
