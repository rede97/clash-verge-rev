# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a fork of [Clash Verge Rev](https://github.com/clash-verge-rev/clash-verge-rev) — a Tauri v2 desktop GUI for Clash Meta proxy client. Monorepo containing a React frontend (TypeScript) and a Rust backend (Tauri app).

**This fork is specifically used to build and publish Windows NSIS user-mode installers** (per-user install, no admin required).

## Work Objective

1. Automatically detect new upstream releases (via `sync-upstream.yml`, runs every 3 hours)
2. Merge upstream code into `main` branch while preserving fork-specific customizations
3. Publish releases **aligned with upstream tags** (same `vX.Y.Z` version numbers, pushed by the sync workflow)
4. Build Windows x64 + ARM64 NSIS user-mode installers via GitHub Actions
5. Publish built installers to **this fork's own** GitHub Releases
6. Serve app auto-updates **exclusively from this fork's Releases** — never from upstream

## Independent Update Channel (Self-Hosted)

The fork is a standalone application regarding updates:

- **Updater endpoint** (`src-tauri/tauri.conf.json`): `https://github.com/rede97/clash-verge-rev/releases/download/updater/update.json` — points to THIS repo, not upstream
- **Signing key**: fork's own Tauri updater keypair (fork's pubkey in `tauri.conf.json`, private key in repo secret `TAURI_SIGNING_PRIVATE_KEY`). Upstream signatures are NOT trusted by this fork
- **Update chain**: tag push → `release.yml` builds NSIS + publishes release → `release-update` job runs `scripts/updater.mjs` → generates `update.json` from the fork's own release assets (`.exe` + `.exe.sig`) → uploads to the `updater` tag release
- **NEVER** change the updater endpoint or pubkey back to upstream values when merging upstream updates

## Branch Strategy

| Branch | Purpose |
|--------|---------|
| `main` | Fork's release branch (also the default branch). Based on upstream `main` + fork customizations. Tags trigger builds. |

> The upstream `dev` mirror branch has been **deleted** from this fork. Only `main` exists. Upstream `dev` (daily autobuild preview channel) is intentionally NOT tracked — this fork only publishes stable releases.

## Automated Workflow

```
sync-upstream.yml (every 3h)
  │
  ├─► Check upstream for new tags not present in this fork
  │
  ├─► No new tag → skip
  │
  └─► New tag found
        │
        ├─► git merge upstream/main → main
        │     │
        │     ├─► Success → push main + tag → triggers release.yml → builds Windows NSIS
        │     │
        │     └─► Conflict → create GitHub Issue (assigned to rede97) → email notification
        │
        └─► release.yml builds only:
              - Windows x64 NSIS (currentUser)
              - Windows ARM64 NSIS (currentUser)
              │
              └─► publish release (body links point to THIS fork's Releases)
                    │
                    └─► release-update job → scripts/updater.mjs
                          → upload update.json to `updater` tag
                          → installed apps update from this fork
```

## Fork-Specific Changes (vs Upstream)

Only **4 files** are modified from upstream. When merging upstream updates, these changes MUST be preserved:

### 1. `src-tauri/tauri.windows.conf.json`
```json
"installMode": "currentUser"  // upstream: "perMachine"
```
**Why**: Enables per-user install without admin privileges.

### 2. `src-tauri/tauri.conf.json`
- Updater plugin enabled with fork's own signing pubkey, single GitHub release endpoint
- `createUpdaterArtifacts` set to `true`

**Why**: This fork uses its own updater signing keys (not upstream's). The endpoint points to this fork's GitHub Releases updater tag.

### 3. `.github/workflows/release.yml`
Complete rewrite from upstream. Key differences:
- **Removed**: macOS builds, Linux builds (amd64/arm64/armv7), Fixed WebView2 builds, Winget submission, Telegram notification, upstream updater signing keys
- **Kept**: Windows x64 + ARM64 NSIS builds only
- **Added**: `release-update` job (runs `pnpm updater` after publishing) to generate `update.json` for this fork's self-hosted update channel
- **Release body**: Only lists Windows user-mode download links, pointing to this fork's Releases (`${{ github.server_url }}/${{ github.repository }}`), never upstream's
- **Branch check**: Tags must be from `main` branch (not `dev`)

### 4. `.github/workflows/autobuild.yml`
- Upstream `schedule` trigger (cron, twice daily) **removed**; only `workflow_dispatch` kept

**Why**: This fork does not publish daily autobuild previews. Keeping the cron would waste Actions minutes and create unwanted `autobuild` releases on the default branch.

## Merge Conflict Resolution Strategy

When merging upstream/main into main produces conflicts, resolve as follows:

### General principle
- **Always keep the fork's change** for any of the 3 modified files
- For all other files, accept the upstream version

### Per-file strategy

| File | Strategy |
|------|----------|
| `tauri.windows.conf.json` | Keep `"currentUser"`, accept all other upstream changes |
| `tauri.conf.json` | Keep fork's own pubkey and updater endpoints, accept all other upstream version/plugin changes |
| `release.yml` | Keep the entire fork version (Windows-only + `release-update` job + fork-only download URLs). If upstream adds new build features (e.g. new attestation step), cherry-pick only those additions into the Windows-only structure |
| `autobuild.yml` | Keep the trigger block WITHOUT `schedule`. Accept all other upstream changes to this file |
| All other files | Accept upstream version unconditionally |

### Example conflict resolution for release.yml
```
CONFLICT: upstream added a new Tauri build parameter "--feature new-thing"

Our version:    args: --target ${{ matrix.target }}
Upstream:       args: --target ${{ matrix.target }} --feature new-thing

Resolution:     args: --target ${{ matrix.target }} --feature new-thing
                (Accept the new parameter, keep our Windows-only matrix)
```

## OpenClaw Automated Conflict Resolution (Planned)

A local OpenClaw instance will:
1. Monitor email for GitHub Issue notifications with title `⚠️ Merge conflict`
2. On receiving notification:
   - Clone/pull the repo locally
   - Fetch upstream main
   - Attempt merge, identify conflicting files
   - Use Claude to resolve conflicts following the strategy above
   - Commit and push resolved code
   - Close the GitHub Issue

## Common Commands

```bash
# Frontend dev server only (port 3000)
pnpm web:dev

# Full Tauri dev (frontend + Rust backend, uses verge-dev feature)
pnpm dev

# TypeScript type checking
pnpm typecheck

# Linting (ESLint + Biome format check)
pnpm lint
pnpm format:check

# Auto-fix linting/formatting
pnpm lint:fix
pnpm format

# Production build (4096MB memory for Node)
pnpm build

# Faster optimized build
pnpm build:fast

# i18n maintenance
pnpm i18n:check          # find unused i18n keys
pnpm i18n:format         # align/cleanup i18n keys (--align --apply)
pnpm i18n:types          # regenerate i18n TypeScript types

# Release scripts
pnpm release:autobuild   # version bump for autobuild
pnpm release:deploytest  # version bump for deploy test

# Rust tests (from workspace root)
cargo test
cargo test -p clash-verge  # app-specific tests

# Rust formatting/linting
cargo fmt
cargo clippy
```

## Architecture

### Frontend (`src/`)
- **Entry**: `src/main.tsx` — bootstraps i18n, theme, preloads app data, sets up React root with providers
- **Routing**: `src/pages/_routers.tsx` — React Router v7 routes
- **Pages**: `src/pages/` — page components (settings, proxies, profiles, etc.)
- **Components**: `src/components/` — shared UI components (MUI-based)
- **Hooks**: `src/hooks/` — custom React hooks
- **Services**: `src/services/` — API calls, Tauri invoke wrappers, i18n, state management
- **Providers**: `src/providers/` — React context providers (app data, window info)
- **Locales**: `src/locales/` — i18n translation JSON files (zh, en, es, ru, ja, ko, fa)
- Path alias: `@/` → `src/`

### Rust Backend (`src-tauri/src/`)
- **Entry**: `main.rs` — sets up custom Tokio runtime (max 16 worker threads), calls `app_lib::run()`
- **`lib.rs`** — Tauri app setup: plugin registration, command generation, event handling (window lifecycle, deep links, tray)
- **`cmd/`** — Tauri IPC commands (invoked from frontend via `tauri-plugin-mihomo-api`)
- **`config/`** — Configuration management: Clash config, Verge app config, profiles, runtime config
- **`core/`** — Core business logic: handle (global state), hotkey, autostart, tray, sysopt, service, notification
  - `core/manager/` — clash config lifecycle (config.rs), state management (state.rs), lifecycle (lifecycle.rs)
- **`enhance/`** — Profile enhancement: field merging, script execution, chain/tun config, builtin JS scripts
- **`feat/`** — Higher-level feature implementations: backup flow, clash lifecycle, icon management, profile operations
- **`module/`** — Optional modules: lightweight mode (macOS), auto backup
- **`process/`** — Async handler utilities (spawn/block_on wrappers)
- **`utils/`** — Helpers: dirs, resolve (scheme URLs, DNS, window), server, tray speed, network, window manager

### Rust Workspace Crates (`crates/`)
- `clash-verge-draft` — Draft/prototype utilities
- `clash-verge-logging` — Custom logging macros and setup
- `clash-verge-signal` — Signal/event broadcasting
- `clash-verge-i18n` — Backend-side internationalization
- `clash-verge-limiter` — Rate limiting
- `tauri-plugin-clash-verge-sysinfo` — System info Tauri plugin

### Key Dependencies
- **Frontend**: React 19, MUI 9, React Router 7, TanStack Query/Table/Virtual, Monaco Editor, dnd-kit, i18next
- **Backend**: Tauri 2.10, Tokio, reqwest, serde_yaml_ng, boa_engine (JS runtime for scripts), warp (local server), sysproxy
- **mihomo IPC**: `tauri-plugin-mihomo` for communicating with the Clash Meta core process via Unix socket

## Build Features (Rust)
- `verge-dev` (default for dev): enables color logging, custom-protocol
- `tauri-dev`: enables devtools plugin
- `tokio-trace`: enables tokio console subscriber for async tracing
- `clippy`: uses mock Tauri context for clippy analysis

## Key Patterns

### Frontend → Backend Communication
Frontend calls Tauri commands via `invoke()` from `@tauri-apps/api`. The `tauri-plugin-mihomo-api` package provides WebSocket IPC for real-time clash status. IPC commands are registered in `lib.rs`'s `generate_handlers()`.

### Config System
Verge config and Clash config are managed via `Config::verge()` and `Config::clash()` async accessors in `src-tauri/src/config/`. Config changes are persisted to JSON files and broadcast via signals.

### Async Runtime
The Tokio runtime handle is set as Tauri's async runtime in `main.rs` before app launch. Use `AsyncHandler::spawn()` and `AsyncHandler::block_on()` from `process/` for async operations.

### Profile Enhancement Pipeline
Profiles go through an enhancement pipeline (`enhance/`): merge → field → chain → tun → script execution. Builtin JavaScript scripts in `enhance/builtin/` are executed via the Boa JS engine.
