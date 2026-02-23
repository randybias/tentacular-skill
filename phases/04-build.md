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

## Part B: Build (one node at a time, strictly in order)

Repeat this cycle for every node, in DAG topological order (roots first):

### Step 1: Write the test fixture

Fixture naming: node file `nodes/fetch-issues.ts` → fixture file `tests/fixtures/fetch-issues.json`. The name before `.ts` is the name before `.json`. Exactly.

Before writing the node, create its fixture:

```json
{
  "input": { /* the exact data structure this node receives */ },
  "expected": { /* the exact data structure this node must return */ }
}
```

The `expected` field MUST assert on the actual data content — not just shape,
not just "non-null". If the node fetches data, the fixture must use a real or
realistic mock output that downstream nodes can use.

### Step 2: Run the test — watch it fail

```bash
tntc test <workflow-name>/<node-name>
```

The test must fail because the node doesn't exist yet. If it passes, the fixture
is wrong — fix the fixture before proceeding.

### Step 3: Write the node

Write the minimum TypeScript to make the test pass.

Node contract:
```typescript
import type { Context } from "tentacular";

export default async function run(ctx: Context, input: unknown): Promise<unknown> {
  // input: validated data from upstream node (or trigger for root nodes)
  // return: data that downstream node(s) will receive — never null, never {}
  return { /* meaningful data */ };
}
```

**Context API — use these, do not invent alternatives:**

```typescript
// Call an external dependency (declared in contract.dependencies)
const dep = ctx.dependency("my-service");
const resp = await dep.fetch!("/path", { method: "GET", headers: { Authorization: `Bearer ${dep.secret}` } });
const data = await resp.json();

// Read a secret (declared in contract.secrets)
const token = ctx.secrets["my-token"];

// Read workflow config (declared in contract.config)
const repoName = ctx.config["repo-name"];

// Logging (does not count as output — never return only log data)
ctx.log.info("message");
ctx.log.error("error message");
```

For full Context API details: `references/node-development.md`.

### Step 4: Run the test — verify it passes AND the output is correct

```bash
tntc test <workflow-name>/<node-name>
```

Verify:
- [ ] Test passes (exit 0)
- [ ] Output matches the `expected` fixture exactly
- [ ] Output is not `{}`, `null`, `undefined`, or an empty array
- [ ] Output contains every field that the next node's data contract lists under `Input`

**Concrete failure test:** take the output object and ask: "Could the next node complete its job using only these fields?" If no → the node is performative. Delete it. Return to Step 1 and redesign.

Examples of outputs that FAIL this test even though they look non-empty:
- `{ status: "ok" }` — next node can't do anything with a status string
- `{ count: 5 }` — next node needed `{ issues: [...] }`, not the count
- `{ message: "fetched 5 issues" }` — a log message dressed as output

### Step 5: Gate before next node

Only write the next node after the current node's test passes AND output is verified.
No batching. No "I'll test them all at the end."

---

## Validate the DAG

After all nodes are written and tested:

```bash
tntc validate
tntc secrets check
```

Both must pass before proceeding to phase 05.

---

## Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Writing all nodes before testing | Cascading failures, unclear which node is broken | One node at a time, test each |
| Fixture `expected: {}` | Test passes even if node returns nothing | Assert on actual data content |
| Node returns `{ status: "ok" }` only | Next node receives no usable data | Return the data the next node needs |
| Skipping DAG data flow map | Performative nodes not caught until pipeline test | Map before coding |
| Writing contract after nodes | Nodes access dependencies not declared in contract | Contract always first |

Proceed to: `phases/05-test-and-deploy.md`.
