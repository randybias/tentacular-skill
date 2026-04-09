# Prompt Metadata (`prompts.yaml`)

## Overview

Every LLM-calling node in a tentacle must have a corresponding entry in
`prompts.yaml`. This file lives alongside `workflow.yaml` in the tentacle
root and declares system prompts, user prompt templates, models, tools,
and output templates.

At deploy time, the builder stores `prompts.yaml` content in a Tier 2
metadata ConfigMap. `wf_describe` returns the parsed prompts and templates,
making them visible to users via Chroma (the enclave UI) and The Kraken
(`show prompt`, `show template` commands).

## Schema

```yaml
version: "1"

prompts:
  - node: <node-filename-without-.ts>    # REQUIRED — must match a node in workflow.yaml
    name: <stable-kebab-case-id>         # REQUIRED — used for iteration and lookup
    description: "<one-line description>"
    model: <model-id>                    # e.g. claude-haiku-4-5, claude-sonnet-4-5
    system_prompt: |
      <full system prompt text>
    user_prompt_template: |
      <user message — use {{input.field}} for dynamic parts>
    tools:                               # only if the node uses tool_use
      - name: <tool-name>
        description: "<what this tool does>"

templates:
  - node: <node-filename-without-.ts>    # REQUIRED
    name: <stable-kebab-case-id>         # REQUIRED
    description: "<what this template produces>"
    format: <markdown|slack-blocks|html|text>
    template: |
      <output template text>
```

## Required fields

- `version` — always `"1"`
- `node` — must match a node filename (without `.ts`) in `workflow.yaml`
- `name` — stable identifier used by Kraken commands and Chroma UI

## When to create entries

**Prompt entry:** for every node that calls an LLM API (Anthropic, OpenAI,
or any model provider). This includes nodes that use tool_use, vision, or
embeddings.

**Template entry:** for every node that produces formatted output — Slack
messages, HTML reports, emails, PDF layouts. If the node constructs a
structured message from data, it has a template.

## User review protocol

After writing an LLM node and its prompts.yaml entry:

1. Present the full system prompt and user prompt template to the user
2. Show the model name and any tools
3. Ask: "Want to adjust the tone, add constraints, or change anything?"
4. Wait for explicit approval before proceeding to testing
5. If changes are requested, update both the node code AND prompts.yaml

This review is **mandatory for every LLM node**. Prompts are the most
business-critical part of an LLM workflow.

## Source of truth

Today, `prompts.yaml` is declarative metadata. The node `.ts` code is the
runtime source of truth — it contains the actual prompt that gets sent to
the LLM. The agent must keep both in sync.

A future engine update will add `ctx.prompt("prompt-name")` so nodes read
directly from `prompts.yaml` at startup. Until then, treat prompts.yaml as
the documentation layer that enables visibility and iteration.

## Metadata pipeline

```
prompts.yaml  ──[tntc deploy]──►  Tier 2 ConfigMap (prompts key)
                                    │
                                    ├──► wf_describe returns prompts[] + templates[]
                                    ├──► Chroma renders prompt text in enclave UI
                                    └──► Kraken `show prompt` / `show template` commands

workflow.yaml ──[tntc deploy]──►  Tier 1 annotations
                                    ├──► tentacular.io/prompt-count
                                    └──► tentacular.io/template-count
```

## Kraken commands

| Command | Description |
|---------|-------------|
| `@kraken show prompts` | List all workflows with prompt counts |
| `@kraken show prompts <workflow>` | List prompts in a specific workflow |
| `@kraken show prompt <workflow> <name>` | Show full prompt text |
| `@kraken show templates` | List all workflows with template counts |
| `@kraken show templates <workflow>` | List templates in a specific workflow |
| `@kraken show template <workflow> <name>` | Show full template text |

Prompts and templates can be looked up by either `name` or `node`.

## Model naming

Use current Anthropic model names:
- `claude-haiku-4-5` — fast, cheap, simple tasks
- `claude-sonnet-4-5` — balanced, most common choice
- `claude-opus-4-5` — most capable, complex analysis
- For OpenAI: `gpt-4o`, `gpt-4o-mini`, etc.

## Size limits

The metadata ConfigMap has a 100KB per-key limit. A single very large
prompts.yaml (many prompts with extensive system prompts) could approach
this. In practice, even the most complex scaffolds (5 prompts, 2 templates)
are well under 10KB.
