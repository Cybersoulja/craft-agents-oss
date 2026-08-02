# CLAUDE.md — craft-agents-oss

Electron + Bun + TypeScript + React desktop app for the Claude Agent SDK. Monorepo.

## Package Manager

**Bun only** — never use npm or yarn here.

```bash
bun install
```

## Dev Commands

```bash
bun run electron:dev            # Run Electron app in dev mode
bun run electron:dev:terminal   # Terminal mode
bun run server:dev              # Backend server only (CRAFT_DEBUG=true)
bun run playground:dev          # UI component playground (port 5173)
bun run viewer:dev              # Session sharing viewer (port 5174)
bun run webui:dev               # Web UI (thin client), port 5175
```

## Validation

```bash
bun test                        # All tests (repo-root bun test + all *.isolated.ts files, run individually)
bun test path/to/some.test.ts   # Single test file
cd packages/shared && bun test tests/llm-connections.test.ts   # Single test, scoped to its package
bun run test:shared:all         # Shared package tests only
bun run typecheck:all           # Type-check every package (run this, not bare `bun run typecheck`, which only checks packages/shared)
bun run lint                    # Lint (IPC sends + electron + shared + ui)
bun run validate:dev            # Full: typecheck + tests + doc-tools
```

Files matching `*.isolated.ts` are excluded from the main `bun test` run and must be run separately (see `test` script in `package.json`) — they likely mutate global/process state that would corrupt other tests in the same run.

## Build & Package

```bash
bun run electron:dist:mac       # Package macOS build
bun run server:build            # Build server binary
bun run electron:build          # Build all Electron assets
```

## Package Layout

| Package                       | Purpose                                        |
| ----------------------------- | ---------------------------------------------- |
| `packages/shared`             | Config, LLM connections, prompts, shared types |
| `packages/core`               | Agent core logic                               |
| `packages/server`             | Backend server entry — also runs headless (see README "Remote Server") |
| `packages/server-core`        | Server core abstractions                       |
| `packages/session-tools-core` | Session/tool abstractions                      |
| `packages/session-mcp-server` | MCP server subprocess                          |
| `packages/pi-agent-server`    | Pi agent server subprocess                     |
| `packages/ui`                 | React UI components (Tailwind + Radix)         |
| `apps/electron`               | Electron shell — desktop GUI (primary), see below |
| `apps/cli`                    | Terminal client — connects to a running server over WebSocket |
| `apps/viewer`                 | Session sharing viewer (standalone Vite app)   |
| `apps/webui`                  | Thin-client web UI (talks to `packages/server` over WS, like the CLI) |

### `apps/electron/src` layout

| Dir          | Purpose                                                        |
| ------------ | ---------------------------------------------------------------- |
| `main/`      | Electron main process — window/menu management, deep links, auto-update, native handlers under `main/handlers/` |
| `preload/`   | Context bridge exposed to the renderer                        |
| `renderer/`  | React UI (Vite + shadcn) — `atoms/` (Jotai state), `pages/`, `components/`, `actions/`, `event-processor/` |
| `shared/`    | Types/schemas shared between main and renderer — `routes.ts`, `route-parser.ts`, `menu-schema.ts`, `feature-flags.ts` |
| `transport/` | IPC/WebSocket transport layer — same protocol used by the desktop app, CLI, and webui thin clients |

Desktop, CLI, and webui are all thin clients over the same RPC/WebSocket protocol (`transport/`); the Electron build just also embeds the server in-process for local-only mode.

## Key Tech

- `@anthropic-ai/claude-agent-sdk` + `@mariozechner/pi-ai` side-by-side — two agent backends, see README "Supported LLM Providers" for which providers route through which
- State: Jotai atoms (`apps/electron/src/renderer/atoms/`)
- Rich text: Tiptap
- IPC: typed sends only — raw `ipcRenderer.send`/`invoke` calls are checked for by `scripts/check-raw-sends.sh` (`bun run lint:ipc-sends`)
- Credentials: AES-256-GCM encrypted file storage in `~/.craft-agent/credentials.enc` (see `packages/shared/src/credentials/`)

## Notes

- `craft-agents-oss-local/` is a separate local worktree for experimental branches
- Sentry error investigation → use the `sentry-investigator` agent
- `CRAFT_DEBUG=true` enables verbose server logging
