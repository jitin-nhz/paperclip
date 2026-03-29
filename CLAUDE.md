# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What Is Paperclip

Paperclip is a control plane for AI-agent companies. It orchestrates agents (Claude, Codex, Cursor, etc.) through a board-governed task system with atomic checkout semantics, cost budgets, and approval gates. The V1 build contract is in `doc/SPEC-implementation.md`. Read `doc/GOAL.md` → `doc/PRODUCT.md` → `doc/SPEC-implementation.md` before making significant changes.

## Commands

```sh
# Development
pnpm install
pnpm dev                  # API + UI with file watching (http://localhost:3100)
pnpm dev:once             # Same, without file watching
pnpm dev:server           # Server only
pnpm dev:ui               # UI only

# Verification (run all before marking work done)
pnpm -r typecheck
pnpm test:run
pnpm build

# Testing
pnpm test:run             # All Vitest tests once
pnpm test                 # Vitest in watch mode
pnpm test:e2e             # Playwright E2E tests
pnpm test:e2e:headed      # E2E tests with browser visible

# Running tests for a single package
cd server && pnpm test
cd ui && pnpm test
cd packages/db && pnpm test

# Database
pnpm db:generate          # Generate migration after schema change
pnpm db:migrate           # Apply pending migrations
pnpm db:backup            # Manual backup

# Health checks
curl http://localhost:3100/api/health
curl http://localhost:3100/api/companies
```

No external PostgreSQL needed in dev — the server uses embedded Postgres persisted at `~/.paperclip/instances/default/db`. Reset it with `rm -rf ~/.paperclip/instances/default/db`.

## Repo Structure

```
server/          Express REST API, orchestration services, WebSocket heartbeats
ui/              React + Vite board UI (Tailwind, Radix UI, TanStack Query)
cli/             paperclipai CLI (Commander.js, interactive onboarding)
packages/
  db/            Drizzle ORM schema + migrations (PostgreSQL)
  shared/        Shared types, constants, validators, API path constants
  adapters/      Agent provider implementations (claude_local, codex_local, cursor_local, etc.)
  adapter-utils/ Utilities shared across adapters
  plugins/       Plugin system and SDK
skills/          Agent skill packages
tests/           Playwright E2E + release smoke tests
doc/             Product and technical documentation
```

## Architecture

The system has three principal layers:

1. **Control plane** (`server/`) — Express 5 REST API with board auth (session) and agent auth (bearer API keys hashed at rest). Manages companies, projects, issues (tasks), and agent runs. All mutations write activity log entries.

2. **Board UI** (`ui/`) — React 19 SPA served by the API server in dev middleware mode. Uses React Router 7, TanStack Query for data fetching, and Lexical/MDXEditor for rich content.

3. **Adapter layer** (`packages/adapters/`) — Each adapter wraps a local LLM agent runtime. Adapters receive wake requests from the server and report heartbeats back. The `packages/adapter-utils/` package provides the shared protocol.

**Data model**: Every entity (issue, project, agent, key, etc.) is company-scoped. Schema lives in `packages/db/src/schema/*.ts`. Shared API types and route constants live in `packages/shared/`.

**Contract sync rule**: Any data model change must propagate through all four layers — `packages/db` schema → `packages/shared` types/validators → `server` routes/services → `ui` API clients and pages.

## Core Invariants to Preserve

- **Single-assignee task model** — an issue has at most one assigned agent at a time
- **Atomic issue checkout semantics** — checkout is a transactional operation
- **Approval gates** — governed actions (e.g., agent join, certain mutations) require board approval
- **Budget hard-stop** — agents are auto-paused when cost budget is exhausted
- **Activity logging** — every mutating route writes an activity log entry
- **Company boundary enforcement** — routes must reject cross-company access; agent keys must not reach other companies

## Database Change Workflow

1. Edit `packages/db/src/schema/*.ts`
2. Export new tables from `packages/db/src/schema/index.ts`
3. `pnpm db:generate` (compiles `packages/db` first, then runs drizzle-kit)
4. `pnpm -r typecheck` to verify

## Lockfile Policy

Do **not** commit `pnpm-lock.yaml` in pull requests. CI on `master` regenerates and commits it automatically.

## PR Message Convention

PRs should start with a "thinking path" — a short top-down narrative from Paperclip's purpose down to the specific change. See `CONTRIBUTING.md` for examples. Include before/after screenshots for UI changes.

## Plan Documents

New design/plan documents belong in `doc/plans/` using `YYYY-MM-DD-slug.md` filenames. Do not replace `doc/SPEC.md` or `doc/SPEC-implementation.md` wholesale — prefer additive updates.

## Worktree Dev

When developing from multiple git worktrees, each needs an isolated Paperclip instance:

```sh
pnpm paperclipai worktree:make <branch-name>   # create worktree + isolated instance in one step
pnpm paperclipai worktree init                 # initialize in an existing worktree
```
