# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Pi Agent Harness monorepo — a self-extensible coding agent CLI with a unified multi-provider LLM API. Published under `@earendil-works` npm scope. Four packages in lockstep versioning (one version for all):

| Package | Path | Purpose |
|---------|------|---------|
| `pi-ai` | `packages/ai` | Unified multi-provider LLM API (~20 providers: Anthropic, OpenAI, Google, Bedrock, Azure, Mistral, Cloudflare, etc.) |
| `pi-agent-core` | `packages/agent` | Agent runtime: tool calling loop, state management, harness, compaction, skills |
| `pi-coding-agent` | `packages/coding-agent` | Interactive CLI (`pi` command): 7 built-in tools, extension system, TUI |
| `pi-tui` | `packages/tui` | Terminal UI framework with differential rendering, editor, components |

## Commands

```bash
npm install --ignore-scripts      # Install deps (never run lifecycle scripts unprompted)
npm run build                     # Build all packages (tui → ai → agent → coding-agent)
npm run check                     # Lint+format (Biome), pinned deps, TS imports, shrinkwrap, type check — run after code changes
./test.sh                         # Run all non-e2e tests (strips API keys, skips LLM-dependent tests)
./pi-test.sh                      # Run pi from source; ./pi-test.sh --no-env strips API keys
```

**Single test** from a package root:
```bash
node ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts
```

**Interactive TUI testing** via tmux:
```bash
tmux new-session -d -s pi-test -x 80 -y 24
tmux send-keys -t pi-test "./pi-test.sh" Enter
sleep 3 && tmux capture-pane -t pi-test -p
tmux send-keys -t pi-test "your prompt" Enter
tmux kill-session -t pi-test
```

## Code Style and Rules

- **TypeScript**: strict mode, ES2022, Node16 modules, `erasableSyntaxOnly` — no enums, namespaces, parameter properties, `import =`, or `export =`. Use explicit fields with constructor assignments.
- **No inline imports**: `await import()`, `import("pkg").Type`, dynamic type imports are forbidden. Top-level imports only.
- **No `any`** unless absolutely necessary.
- **Biome**: tabs, 3-width indent, 120 line width. Run `npm run check` after changes.
- **No emojis** in commits, issues, PRs, or code.
- **Never modify** `packages/ai/src/models.generated.ts` directly; edit `packages/ai/scripts/generate-models.ts` and regenerate.
- **Never run** `npm run build` or `npm test` unless explicitly asked.
- **Never run** full vitest suite directly (contains e2e tests activated by env vars). Use `./test.sh` or run specific test files.
- **Commit format**: `{feat,fix,docs}[(ai,tui,agent,coding-agent)]: message`
- **Only stage** files you changed in this session; never `git add -A` / `git add .`.
- **Never destructive git**: `git reset --hard`, `git checkout .`, `git clean -fd`, `git stash`, `git commit --no-verify`.
- **Direct external deps** pinned exact; treat lockfile changes as reviewed code. Pre-commit blocks lockfile commits unless `PI_ALLOW_LOCKFILE_CHANGE=1`.

## Architecture

### Dependency graph (build order)
`tui` → `ai` → `agent` → `coding-agent`

### Agent loop pattern
`pi-agent-core` provides a generic loop: prompt → model response → tool calls → tool results → next prompt. The coding agent wires this with its 7 built-in tools (read, bash, edit, write, grep, find, ls).

### Provider abstraction (`packages/ai`)
Each provider is a standalone file under `providers/`. A generated model catalog (`models.generated.ts`) maps model IDs to providers and pricing. Providers register lazily via `register-builtins.ts`. The faux provider (`providers/faux.ts`) is a mock for testing without real LLM calls.

### Extension system (`packages/coding-agent/src/core/extensions/`)
Hook points: beforeAgentStart, beforeProviderRequest, toolCall, input, session, etc. Extensions add custom tools, commands, UI widgets, and skill blocks. Discovered from `.pi/extensions/` or npm packages.

### Tool factory pattern
Each tool exports `createXxxToolDefinition` (for the harness) and `createXxxTool` (for execution), created per-session with a `cwd` parameter.

### Multi-mode execution
The coding agent runs in 3 modes: interactive (TUI), RPC (JSONL protocol for external integrations), and print (one-shot pipe).

### Session management
Sessions stored as JSONL with compaction (context summarization). Versioned format with migration system.

## Testing Conventions

- **Suite tests** (`packages/coding-agent/test/suite/`): use `test/suite/harness.ts` + faux provider. No real APIs, keys, or paid tokens.
- **Regression tests**: under `test/suite/regressions/` named `<issue-number>-<short-slug>.test.ts`.
- **AI provider tests** (`packages/ai/test/`): each new provider must be added to `stream.test.ts` and the broader test matrix (tokens, abort, empty, context-overflow, unicode-surrogate, tool-call-without-result, image-tool-result, total-tokens, cross-provider-handoff).
- See `.pi/skills/add-llm-provider.md` for the full provider addition checklist.

## Adding a New LLM Provider

Follow `.pi/skills/add-llm-provider.md` — covers: core types (`types.ts`), provider implementation (`providers/`), lazy registration, model generation script, test matrix, coding-agent wiring (model-resolver, provider-display-names, args), and docs.

## Releasing

Lockstep versioning: all packages share one version. `patch` = fixes+additions, `minor` = breaking changes. No major releases.

1. Update CHANGELOGs (run `/cl` prompt first if needed).
2. Local smoke test: `npm run release:local -- --out /tmp/pi-local-release --force`.
3. Release: `PI_ALLOW_LOCKFILE_CHANGE=1 npm_config_min_release_age=0 npm run release:patch` (or `:minor`).
4. CI publishes npm on tag push — no local `npm publish` needed.
