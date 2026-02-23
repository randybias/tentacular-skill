# Phase 04: Build a Workflow

**Prerequisites: phases 01–03 complete. Profile read. Agent Guidance applied.**

## Part A: Design (before writing any code)

### 1. Name and scaffold

```bash
tntc init <workflow-name>   # kebab-case required
cd <workflow-name>
```

### 2. Author the contract first

Every external dependency must be declared in `workflow.yaml` under `contract.dependencies`
before any node is written. No exceptions.

For contract structure: `references/contract.md`.

### 3. Map the full data flow

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

### 4. Get confirmation

Present the DAG, data flow map, and contract to the user. Get explicit confirmation
before writing any code.

---

## Part B: Build (one node at a time, strictly in order)

Repeat this cycle for every node, in DAG topological order (roots first):

### Step 1: Write the test fixture

Before writing the node, write its test fixture in `tests/fixtures/<node-name>.json`:

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

### Step 4: Run the test — verify it passes AND the output is correct

```bash
tntc test <workflow-name>/<node-name>
```

Verify:
- [ ] Test passes (exit 0)
- [ ] Output matches the `expected` fixture exactly
- [ ] Output is not `{}`, null, or empty
- [ ] Output contains the fields the next node's input expects

**If the output is empty or only contains status/log fields: the node is
performative. Delete it. Return to Step 1 and redesign.**

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
