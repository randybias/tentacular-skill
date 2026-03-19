# Node Contract Reference

Node signature, Context API, dependency access patterns, and common mistakes.
For test-driven node development, see `phases/04-build.md`.

## Node Signature

Every node is a TypeScript file with a single default export:

```typescript
import type { Context } from "tentacular";

export default async function run(
  ctx: Context,
  input: unknown
): Promise<unknown> {
  // input: output from upstream node(s) via edges
  // return value: passed to downstream node(s)
  ctx.log.info("processing");
  return { result: "done" };
}
```

- `input` is the output from upstream node(s) connected via `edges:`.
- The return value is passed to downstream node(s) via their `input`.
- Nodes MUST return meaningful data. Returning `{}` or `{ status: "ok" }`
  is a contract violation -- downstream nodes cannot act on empty output.

## Context API

| Member | Type | Description |
|--------|------|-------------|
| `ctx.dependency(name)` | `(string) => DependencyConnection` | Primary API for external services. Returns connection metadata (host, port, protocol, authType, protocol-specific fields) and resolved secret value. Throws if dependency is not declared in the contract. |
| `ctx.log` | `Logger` | Structured logging with `info`, `warn`, `error`, `debug` methods. All output prefixed with `[nodeId]`. |
| `ctx.config` | `Record<string, unknown>` | Workflow-level config from `config:` in workflow.yaml. Use for business-logic parameters only (e.g., `target_repo`). |
| `ctx.fetch` | legacy | **LEGACY -- do not use.** HTTP request with auto URL construction. Use `ctx.dependency()` instead. |
| `ctx.secrets` | legacy | **LEGACY -- do not use.** Direct secret access. Use `ctx.dependency()` instead. |

## Using ctx.dependency()

Nodes access external service connection info through the contract dependency
API:

```typescript
// PostgreSQL dependency
const pg = ctx.dependency("postgres");
// pg.protocol, pg.host, pg.port, pg.database, pg.user
// pg.authType -- "password", pg.secret -- resolved value

// HTTPS dependency
const gh = ctx.dependency("github-api");
// gh.fetch(path, init?) -- URL builder (no auth injection)
const resp = await gh.fetch!("/repos/org/repo", {
  headers: { "Authorization": `Bearer ${gh.secret}` },
});
```

**Auth is explicit.** `dep.fetch()` builds the URL
(`https://<host>:<port><path>`) but does not inject auth headers. Nodes set
auth headers themselves using `dep.secret` and `dep.authType`.

In mock context (during `tntc test`), `ctx.dependency()` returns mock values
and records access for drift detection.

**Migration note:** Replace `ctx.config` + `ctx.secrets` assembly with
`ctx.dependency()`. Connection metadata and secret references belong in
`contract.dependencies`. The `config` section should contain only
business-logic parameters (e.g., `target_repo`, `sep_label`).

## Common Node Mistakes

These mistakes are caught during testing but are worth knowing up front:

- **Using `ctx.fetch()` instead of `ctx.dependency()`** -- legacy API,
  flagged as contract violation when a contract is present.
- **Using `ctx.secrets` instead of `ctx.dependency()`** -- same issue.
- **Returning `{}` or `{ status: "ok" }`** -- performative node anti-pattern.
  Downstream nodes receive empty data. Return the actual result.
- **Not handling auth explicitly** -- `dep.fetch()` builds the URL but does
  not inject auth headers. Always pass auth using `dep.secret`.
- **Not declaring the dependency in the contract** -- `ctx.dependency("foo")`
  throws at runtime if `foo` is not in `contract.dependencies`. Declare all
  dependencies before writing node code.
- **Assuming `input` shape without checking** -- the first node in a DAG
  receives trigger input (may be `null` or a small JSON object). Guard
  accordingly.

## See Also

- `phases/04-build.md` -- test-driven node development, mock context setup,
  fixture format, auth testing patterns.
- `references/contract-model.md` -- full contract specification, exoskeleton
  services, mixed dependencies.
- `references/workflow-spec.md` -- workflow.yaml format, edges, node `path:`
  field.
