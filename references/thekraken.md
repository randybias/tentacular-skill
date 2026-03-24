# The Kraken v2 -- Slack Bot for Tentacular

## Overview

The Kraken is a Slack bot that provides a natural-language conversational
interface to the Tentacular platform. Users interact with an AI assistant
in Slack channels and threads; the assistant can manage tentacles, query
cluster state, schedule tasks, and perform general-purpose work using
Claude Code's full tool set.

The Kraken v2 replaces the original Docker-in-Docker (DinD) execution model
with in-process Claude Agent SDK `query()` calls. This eliminates container
image builds, Docker socket dependencies, and cold-start latency. The agent
runs directly in the Node.js host process.

Repository: `thekraken2` (private, `github.com/randybias/thekraken2`)

## Architecture

```
Slack (Socket Mode)
        |
  Slack Bolt event handlers
        |
  Orchestrator (src/index.ts)
        |
  +-----------+     +------------------+
  | SQLite DB |     | Group Queue      |
  | (messages,|     | (per-channel and |
  |  sessions,|     |  per-thread      |
  |  tasks)   |     |  concurrency)    |
  +-----------+     +------------------+
                          |
                    Agent Manager (src/agent-manager.ts)
                          |
                    Claude Agent SDK query()
                          |
              +-----------+-----------+
              |                       |
        MCP: kraken             MCP: tentacular
        (stdio, tools:          (HTTP, tools:
         send_message,           wf_list, ns_list,
         schedule_task,          wf_run, etc.)
         list/pause/resume/
         cancel/update_task)
```

### Key Components

| File | Purpose |
|------|---------|
| `src/index.ts` | Orchestrator: Slack connection, message loop, routing |
| `src/agent-manager.ts` | In-process agent execution via SDK `query()` |
| `src/agent-tools.ts` | Stdio MCP server providing Kraken-specific tools |
| `src/channels/slack.ts` | Slack Bolt channel (Socket Mode) |
| `src/db.ts` | SQLite for messages, sessions, tasks, groups |
| `src/group-queue.ts` | Per-slot concurrency with idle timeout |
| `src/task-scheduler.ts` | Cron/interval/once task execution |
| `src/ipc.ts` | File-based IPC between agent tools and host |
| `src/config.ts` | Environment-driven configuration |
| `src/health.ts` | HTTP health endpoint for Kubernetes probes |

### Agent Execution Model

The agent manager uses a `MessageStream` (async iterable) as the prompt
source, keeping `isSingleUserTurn=false`. This is critical: a plain string
prompt would cause the SDK to close stdin after the first result, killing
any subagents mid-execution.

The stream pattern also enables follow-up messages: when new Slack messages
arrive while the agent is active, they are piped directly into the running
query session rather than queuing for a new invocation.

Each agent invocation receives:
- **System prompt**: Claude Code preset, with global CLAUDE.md appended
  for non-main groups
- **Working directory**: `groups/{group-folder}/` (per-group isolated
  filesystem with CLAUDE.md memory)
- **Session resume**: Prior session ID for conversation continuity
- **Permission mode**: `bypassPermissions` (agent has full tool access)
- **MCP servers**: `kraken` (stdio, Slack-specific tools) and optionally
  `tentacular` (HTTP, cluster operations)
- **Allowed tools**: Bash, Read, Write, Edit, Glob, Grep, Task, TeamCreate,
  SendMessage, ToolSearch, Skill, plus all MCP tools

### Thread Support

Slack threads get independent conversation contexts:
- Each thread has its own session ID (stored via `thread_ts` key)
- Threads run in their own queue slot, concurrent with the parent channel
- Queue key format: `{chatJid}|thread|{thread_ts}` for threads,
  `{chatJid}` for top-level channel messages

### MCP Tool Servers

**kraken** (stdio, spawned per invocation):
Tools for Slack-native operations. The agent tools process receives context
via environment variables (`KRAKEN_CHAT_JID`, `KRAKEN_GROUP_FOLDER`,
`KRAKEN_IS_MAIN`, `KRAKEN_THREAD_TS`) and communicates back to the host
via atomic file-based IPC.

| Tool | Purpose |
|------|---------|
| `send_message` | Send a message to Slack channel/thread immediately |
| `schedule_task` | Schedule a recurring or one-time task |
| `list_tasks` | List scheduled tasks (scoped to group or all) |
| `pause_task` | Pause a scheduled task |
| `resume_task` | Resume a paused task |
| `cancel_task` | Cancel and delete a task |
| `update_task` | Modify an existing task's prompt or schedule |

**tentacular** (HTTP, optional):
The Tentacular MCP server connection, configured via `TENTACULAR_MCP_URL`
and `TENTACULAR_MCP_TOKEN` environment variables. Provides all standard
Tentacular MCP tools (`wf_list`, `ns_list`, `wf_run`, `audit_rbac`, etc.)
for cluster operations through the agent.

## Deployment

The Kraken v2 runs as a single Kubernetes pod (no DinD sidecar). Key
deployment artifacts:

- **PVC**: Persistent storage for SQLite database, group folders, and
  session data
- **NetworkPolicy**: Egress to Slack API (`wss://wss-primary.slack.com`,
  `https://slack.com/api/`), Anthropic API, and the Tentacular MCP server
- **Secrets**: Slack Bot Token, Slack App Token, Claude authentication
  (OAuth token or API key), optional Tentacular MCP token
- **Health endpoint**: HTTP `/healthz` for liveness/readiness probes

### Configuration

| Variable | Default | Purpose |
|----------|---------|---------|
| `ASSISTANT_NAME` | `Kraken` | Trigger word and response prefix |
| `SLACK_BOT_TOKEN` | (required) | Slack Bot User OAuth Token |
| `SLACK_APP_TOKEN` | (required) | Slack App-Level Token (Socket Mode) |
| `CLAUDE_CODE_OAUTH_TOKEN` | (one required) | Claude subscription auth |
| `ANTHROPIC_API_KEY` | (one required) | Claude API key auth |
| `TENTACULAR_MCP_URL` | (optional) | Tentacular MCP server URL |
| `TENTACULAR_MCP_TOKEN` | (optional) | Tentacular MCP bearer token |
| `MAX_CONCURRENT_AGENTS` | `5` | Max parallel agent invocations |
| `AGENT_IDLE_TIMEOUT` | `1800000` | Idle timeout before closing agent (ms) |
| `AGENT_TIMEOUT` | `1800000` | Max agent execution time (ms) |
| `TZ` | system | Timezone for scheduled tasks |

### Memory System

Hierarchical CLAUDE.md files provide persistent memory:
- `groups/global/CLAUDE.md` -- shared context loaded for all groups
- `groups/{folder}/CLAUDE.md` -- per-group/channel memory
- `groups/{folder}/*.md` -- files created by the agent during conversation

The main group has elevated privileges: it can write to global memory,
manage group registrations, and schedule tasks for other groups.

## Differences from NanoClaw

The Kraken v2 is forked from NanoClaw but differs in several key ways:

| Aspect | NanoClaw | The Kraken v2 |
|--------|----------|---------------|
| Execution | Docker container per invocation | In-process SDK `query()` |
| Channel | Multi-channel (WhatsApp, Telegram, etc.) | Slack-only |
| Deployment | macOS launchd / systemd | Kubernetes pod |
| MCP integration | None | Tentacular MCP for cluster ops |
| Thread model | No thread support | Per-thread conversation context |
| Identity | Personal assistant | Tentacular platform bot |
| Credential mgmt | OneCLI gateway | Kubernetes Secrets |
