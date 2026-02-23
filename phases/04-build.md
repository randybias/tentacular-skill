# Phase 04: Build a Workflow

**Prerequisites: phases 01–03 complete. Profile read. Agent Guidance applied.**

## Part A: Design (before writing any code)

### 1. Scaffold in the right place

```bash
cd ~/workspace/tentacles
tntc init <workflow-name>   # kebab-case required
cd <workflow-name>
```

See Directory Rules in SKILL.md. Secrets (`.secrets.yaml`) live here and are never committed.

**workflow.yaml schema — two common mistakes:**

Nodes use `path:` (not `source:`). Edges are a separate top-level `edges:` list
(not `depends_on:` inside node blocks). The init scaffold shows the correct format
but the wrong pattern is easy to write from memory:

```yaml
# ✅ CORRECT
nodes:
  fetch-data:
    path: ./nodes/fetch-data.ts
  transform:
    path: ./nodes/transform.ts

edges:
  - from: fetch-data
    to: transform

# ❌ WRONG — these keys do not exist in the schema
nodes:
  fetch-data:
    source: nodes/fetch-data.ts      # ← wrong key
  transform:
    source: nodes/transform.ts
    depends_on:                       # ← wrong: edges go in their own section
      - fetch-data
```

Run `tntc validate` after writing `workflow.yaml` to catch schema errors before writing any nodes.

### 2. Choose your trigger type

Triggers are defined in `workflow.yaml` under `triggers:`. Common types:

| Type | Use when |
|------|----------|
| `manual` | Triggered via `POST /run` or `tntc run` |
| `cron` | Scheduled execution — add `schedule: "0 9 * * *"` |
| `queue` | NATS subject subscription — add `subject: my.subject` |

For full trigger specs and options: `references/workflow-spec.md`.

### 3. Author the contract first

Every external dependency must be declared in `workflow.yaml` under `contract.dependencies`
before any node is written. No exceptions.

For contract structure: `references/contract.md`.
For agent planning steps and E2E cycle: `references/agent-workflow.md`.

### 4. Map the full data flow

For every node in the DAG, define its **data contract** before writing a single line:

```
Node: <node-name>
  Input:  <describe the data structure this node receives>
  Action: <what this node does to the data>
  Output: <describe the data structure this node returns>
  Edge:   <which node(s) consume this output>
```

Do this for every node. Trace the complete path from root input to final output.

**Check for performative nodes before writing any code.**
A node is performative (forbidden) if any of these are true:
- Its output is `{}`, `null`, `undefined`, or only log/status fields
- No downstream node consumes its output
- Removing it from the DAG does not change the final result
- It does not use any field from its input

If a node in your design is performative → redesign the DAG. Do not proceed.

For the full performative node definition and examples: `references/node-data-flow.md`.

### 5. Get confirmation

Present the DAG, data flow map, and contract to the user.

**STOP. Do not write any code. Do not create any files. Do not begin Part B.**

Wait for the user to reply with explicit confirmation that the design is correct.
A response of "looks good", "go ahead", or "yes" counts. Silence does not count.
Starting to write nodes before confirmation is a violation of Iron Law 4.

---

## Part B: Build

Nodes can be written in any order or in parallel. What is not negotiable:
**every node must be individually tested and its credentials verified before
the DAG is assembled and deployed.** Writing a node is not done until its test passes.

### For each node:

#### Step 1: Write the test fixture

Fixture naming: node file `nodes/fetch-issues.ts` → fixture file `tests/fixtures/fetch-issues.json`. The name before `.ts` is the name before `.json`. Exactly.

```json
{
  "input": { /* the exact data structure this node receives */ },
  "expected": { /* the exact data structure this node must return */ }
}
```

The `expected` field MUST assert on actual data content — not just shape, not just "non-null". If the node fetches data, the fixture must use a real or realistic mock output that downstream nodes can use.

#### Step 2: Verify the fixture fails first

```bash
tntc test <workflow-name>/<node-name>
```

Must fail because the node doesn't exist. If it passes: the fixture is wrong — fix it.

#### Step 3: Write the node

```typescript
import type { Context } from "tentacular";

export default async function run(ctx: Context, input: unknown): Promise<unknown> {
  // input: data from upstream node (or trigger for root nodes)
  // return: data downstream nodes will receive — never null, never {}
  return { /* data, not status */ };
}
```

**Context API:**

```typescript
// External dependency (must be declared in contract.dependencies)
const dep = ctx.dependency("my-service");
const resp = await dep.fetch!("/path", { method: "GET", headers: { Authorization: `Bearer ${dep.secret}` } });
const data = await resp.json();

// Secrets and config
const token = ctx.secrets["my-token"];
const repoName = ctx.config["repo-name"];

// Logging — never return only log data
ctx.log.info("message");
```

For full Context API: `references/node-development.md`.

#### Step 4: Run the test — verify it passes AND output is correct

```bash
tntc test <workflow-name>/<node-name>
```

- [ ] Test passes (exit 0)
- [ ] Output is not `{}`, `null`, `undefined`, or an empty array
- [ ] Output contains every field the next node's data contract lists under `Input`

**Concrete failure test:** ask "Could the next node complete its job using only these fields?" If no → the node is performative. Redesign.

Outputs that FAIL this test:
- `{ status: "ok" }` — next node can't use a status string
- `{ count: 5 }` — next node needed `{ issues: [...] }`, not the count
- `{ message: "fetched 5 issues" }` — a log string is not data

#### Step 5: Validate credentials for any node that touches an authenticated dependency

This applies to: external APIs, databases, message queues, object storage, caches,
or any other dependency that requires auth.

An auth failure in a node test means the node is not done. **Do not proceed.**

Signs of a broken credential:
- HTTP 401 or 403 from an API
- Database connection refused or authentication failed
- Queue consumer/producer rejected (permission denied, bad credentials)
- Any "unauthorized", "forbidden", or "invalid token" error in test output

```bash
# Stop. Surface the failure to the user.
# Ask for the correct credentials, connection string, or subscription key.
# Do not push the DAG with a node that cannot authenticate to its dependency.
```

A node is only "done" when:
- Its individual test passes
- Every authenticated dependency it uses responds successfully

---

## Gate: all nodes done before DAG assembly

Before running `tntc test --pipeline` or deploying:

- [ ] Every node has a passing individual test
- [ ] Every node that touches an authenticated dependency has had its credentials verified (no auth errors)
- [ ] No node's test was skipped on the assumption it would be caught later

```bash
tntc validate
tntc secrets check
```

Both must pass before proceeding to phase 05.

---

## Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Auth failure in node test ignored (403, connection refused, bad token) | Broken auth lands in prod | Stop, ask user for correct credentials before continuing |
| Fixture `expected: {}` | Test passes even if node returns nothing | Assert on actual data content |
| Node returns `{ status: "ok" }` only | Next node receives no usable data | Return the data the next node needs |
| Pushing DAG before pipeline test | Broken data flow lands in dev/prod | `tntc test --pipeline` must pass first |
| Skipping DAG data flow map | Performative nodes not caught until pipeline test | Map before coding |
| Writing contract after nodes | Nodes access dependencies not declared in contract | Contract always first |
