# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

For the full repository guidelines (commit workflow, PR merge flow, release process, VM ops, agent-specific notes), see [AGENTS.md](AGENTS.md).

## Build, Lint, Test Commands

```bash
pnpm install                  # Install deps (pnpm 10.23.0, Node 22+)
pnpm build                    # Type-check + compile TypeScript to dist/
pnpm lint                     # Oxlint (type-aware)
pnpm lint:fix                 # Auto-fix lint + formatting
pnpm format                   # Check formatting (oxfmt)
pnpm format:fix               # Auto-fix formatting
pnpm test                     # Unit/integration tests (Vitest)
pnpm test:coverage            # Tests with V8 coverage report
pnpm test:e2e                 # E2E tests (gateway, WS/HTTP, pairing)
pnpm test:live                # Live tests against real providers (costs $, needs API keys)
pnpm test:watch               # Watch mode
```

Run a single test file: `vitest run src/path/to/file.test.ts`

Pre-push gate: `pnpm lint && pnpm build && pnpm test`

Pre-commit hooks: `prek install` (runs oxlint, oxfmt, detect-secrets, shellcheck, actionlint)

### Development

```bash
pnpm openclaw ...             # Run CLI commands (TypeScript via tsx)
pnpm dev                      # Run TypeScript directly
pnpm gateway:dev              # Dev gateway (skip channels)
pnpm gateway:watch            # Watch mode with auto-rebuild on TS changes
pnpm tui                      # Terminal UI
pnpm ui:dev                   # Web UI dev mode
```

### Platform Builds

```bash
pnpm mac:package              # Package macOS app
pnpm ios:build / ios:gen / ios:open / ios:run   # iOS
pnpm android:assemble / android:install / android:run  # Android
```

## Architecture Overview

**OpenClaw** is a personal AI assistant that runs locally as a gateway, connecting to 28+ messaging channels via a unified interface.

### Core Architecture Layers

- **Gateway** (`src/gateway/`) — WebSocket control plane for sessions, channels, config, tools. Single entry point for all client connections.
- **Channels** (`src/channels/`, `src/routing/`) — Unified message interface. Built-in channels in `src/` (telegram, discord, slack, signal, whatsapp, imessage, line, web). Extension channels in `extensions/` (msteams, matrix, zalo, googlechat, twitch, mattermost, etc).
- **Agents** (`src/agents/`) — Agent runtime, auth profiles, tools, streaming. Uses Pi agent libraries (`@mariozechner/pi-agent-core`, `@mariozechner/pi-ai`).
- **Sessions** (`src/sessions/`) — Session state management. Main (direct) + groups with activation/queue modes.
- **Providers** (`src/providers/`) — Model provider adapters (Anthropic, OpenAI, Bedrock, etc).
- **Plugin System** (`src/plugins/`, `src/plugin-sdk/`) — Extension loading and SDK. Runtime resolves `openclaw/plugin-sdk` via jiti alias.
- **CLI** (`src/cli/`, `src/commands/`) — CLI entry point, commands, profiles. Entry: `openclaw.mjs` → `src/entry.ts`.

### Key Subsystems

| Directory | Purpose |
|-----------|---------|
| `src/media/`, `src/media-understanding/` | Unified media pipeline (images, audio, video) across all channels |
| `src/browser/` | Browser control tools (Chromium/Chrome) |
| `src/canvas-host/` | Canvas rendering (A2UI) |
| `src/hooks/` | Auth, message, presence hooks for extensibility |
| `src/cron/` | Cron job scheduling |
| `src/tts/` | Text-to-speech (node-edge-tts) |
| `src/tui/` | Terminal UI (Pi TUI) |
| `src/security/` | Rate limits, validators |
| `src/memory/` | Persistence layer |
| `src/link-understanding/` | URL/link parsing & preview |

### Monorepo Layout

- **Root** — Main `openclaw` CLI + core
- **`ui/`** — Web UI (separate tsconfig)
- **`packages/`** — Monorepo packages (clawdbot, moltbot)
- **`extensions/`** — 28+ channel/feature plugins (workspace packages)
- **`apps/`** — Native apps: macOS (SwiftUI), iOS (SwiftUI), Android (Kotlin), shared (OpenClawKit)
- **`docs/`** — Mintlify documentation site

Workspace config: `pnpm-workspace.yaml`. Tests colocated as `*.test.ts`, e2e as `*.e2e.test.ts`, live as `*.live.test.ts`.

## Coding Conventions

- **TypeScript ESM** with strict mode. Avoid `any`.
- **Oxlint + Oxfmt** for linting/formatting. Run before commits.
- Keep files under ~500 LOC when feasible (`pnpm check:loc`).
- Use existing patterns for CLI options and DI via `createDefaultDeps`.
- CLI progress: use `src/cli/progress.ts` (osc-progress + @clack/prompts spinner).
- Terminal output: use `src/terminal/palette.ts` for colors (no hardcoded ANSI), `src/terminal/table.ts` for tables.
- Naming: **OpenClaw** for product/docs, `openclaw` for CLI/package/config keys.

## Critical Constraints

- **Commits**: Use `scripts/committer "<msg>" <file...>` to scope commits. Never manual `git add`/`git commit`.
- **Multi-agent safety**: Do not create/apply/drop git stash, do not create/modify git worktrees, do not switch branches — unless explicitly requested. Scope commits to your changes only.
- **Dependencies**: Never update the Carbon dependency. Any `pnpm.patchedDependencies` entry must use exact versions (no `^`/`~`). Patching requires explicit approval.
- **Plugin deps**: Keep plugin-only deps in the extension `package.json`, not root. Avoid `workspace:*` in `dependencies`.
- **Node modules**: Never edit `node_modules` — updates overwrite.
- **Tool schemas**: Avoid `Type.Union` / `anyOf`/`oneOf`/`allOf` in tool input schemas. Avoid raw `format` property names. Use `stringEnum`/`optionalStringEnum` for string lists.
- **Messaging channels**: When refactoring shared logic (routing, allowlists, pairing), consider all built-in + extension channels.
- **Version locations**: `package.json`, `apps/android/app/build.gradle.kts`, `apps/ios/Sources/Info.plist`, `apps/macos/Sources/OpenClaw/Resources/Info.plist`, `docs/install/updating.md`.
- **Docs links**: Internal doc links are root-relative, no `.md`/`.mdx` extension. Avoid em dashes and apostrophes in headings (breaks Mintlify anchors). Docs content must be generic — no personal hostnames/paths.

## Testing Notes

- Coverage thresholds: 70% lines/functions/statements, 55% branches.
- Live tests need API keys (from `~/.profile`): `OPENCLAW_LIVE_TEST=1 pnpm test:live`.
- Docker E2E: `pnpm test:docker:all`.
- Pure test additions don't need changelog entries unless they change user-facing behavior.
- Prefer real connected devices over simulators/emulators for mobile testing.