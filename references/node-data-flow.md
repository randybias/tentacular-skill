# Node Data Flow Reference

## The Rule

**Every node must receive data, transform or act on it, and return data.**

No node exists only to log, to mark progress, or to call a side-effect without
passing something meaningful downstream. A node that does not carry data forward
is performative — it is forbidden.

## Performative Node Definition

A node is **performative** (forbidden) if ANY of the following are true:

1. It returns `{}`, `null`, `undefined`, or a value with zero fields consumed by the next node
2. Its return value is only log/status metadata (e.g. `{ status: "ok", message: "done" }`)
   and downstream nodes don't use those fields
3. Removing it from the DAG would not change the final workflow output
4. It ignores its input entirely (doesn't use any field from the data it receives)
5. Its sole purpose is "signalling" — letting another node know something happened

## Data Flow Contract (define before writing any node)

```
Node: <name>
  Input:  <type/shape of data received — where does it come from?>
  Action: <what does this node do with the input?>
  Output: <type/shape of data returned — what fields does the next node need?>
  Edges:  <which node(s) receive this output?>
```

Complete this for every node before writing any code. Trace from root to terminal:
root input → node A output → node B input → node B output → ... → final result.

If any node in the chain has "Output: none" or "Output: status only" — the chain
is broken. Redesign before coding.

## Examples

### ❌ Performative node (forbidden)

```typescript
// BAD: fetches data but returns only a status
export default async function run(ctx: Context, input: unknown): Promise<unknown> {
  const data = await ctx.dependency("github-api").fetch!("/repos/org/repo/issues");
  ctx.log.info("fetched issues");
  return { status: "fetched", count: (await data.json()).length };
}
// The next node gets count but not the actual issues — it can't do its job.
```

```typescript
// BAD: side-effect only, no data passed on
export default async function run(ctx: Context, input: unknown): Promise<unknown> {
  await ctx.dependency("slack").fetch!("/chat.postMessage", {
    method: "POST",
    body: JSON.stringify({ text: "Pipeline started" }),
  });
  return {};
  // Downstream nodes receive {} — useless
}
```

### ✅ Non-performative nodes (correct)

```typescript
// GOOD: transforms and passes full data forward
export default async function run(ctx: Context, input: { issues: Issue[] }): Promise<unknown> {
  const openIssues = input.issues.filter(i => i.state === "open");
  return { issues: openIssues, count: openIssues.length, fetchedAt: new Date().toISOString() };
  // Next node receives the filtered issues it needs
}
```

```typescript
// GOOD: side-effect node that also passes data forward
export default async function run(ctx: Context, input: { summary: string; items: Item[] }): Promise<unknown> {
  const slack = ctx.dependency("slack");
  await slack.fetch!("/chat.postMessage", {
    method: "POST",
    body: JSON.stringify({ text: input.summary }),
    headers: { Authorization: `Bearer ${slack.secret}` },
  });
  // Return the data downstream nodes need, not just the notification result
  return { notified: true, summary: input.summary, items: input.items };
}
```

## Terminal Nodes

A **terminal node** (no outgoing edges) is exempt from the "passes data forward"
requirement — but it MUST still return a meaningful result. The final output of a
workflow is what `tntc run` returns. It must contain the actual result of the
workflow's purpose, not a status string.

```typescript
// BAD terminal node
return { status: "complete" };

// GOOD terminal node
return { report: generatedReport, processedCount: n, errors: [] };
```

## Detecting a Broken Data Chain

Run the pipeline test and inspect the output:

```bash
tntc test --pipeline
```

Then run live and inspect:

```bash
tntc run <workflow-name>
```

Signs of a broken chain:
- Final output is `{}` or `{ status: "ok" }`
- A node's test passes but pipeline test fails — the fixture didn't check real data shape
- Output is present but downstream node ignores it — the edge is wrong

## Fixing a Performative Node

1. Identify which field the downstream node actually needs
2. Trace back to where that data was last available (which node fetched or computed it)
3. Either:
   a. Have the upstream node include that field in its return value, OR
   b. Remove the performative node and move its side-effect into the node that
      has the data (combine steps where it makes the chain cleaner)
4. Update the data flow contracts for affected nodes
5. Update the test fixtures for affected nodes
6. Re-run affected node tests AND the full pipeline test

Do not add a passthrough field (`return { ...input, status: "ok" }`) just to
satisfy the rule. If a node needs to add a passthrough, question whether it
belongs in the DAG at all.
