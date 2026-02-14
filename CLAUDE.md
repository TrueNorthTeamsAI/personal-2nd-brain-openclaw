# OpenCLAW (Fork)

This is a fork of [openclaw/openclaw](https://github.com/openclaw/openclaw) — a multi-channel AI gateway with extensible messaging integrations. It connects LLMs to WhatsApp, Telegram, Discord, Slack, Signal, iMessage, MS Teams, and more.

**Upstream**: `openclaw/openclaw` (GitHub)
**Fork**: `TrueNorthTeamsAI/personal-2nd-brain-openclaw` (GitHub)

## Tech Stack

- **Runtime**: Node.js 22 (ESM)
- **Language**: TypeScript (tsdown build)
- **Package Manager**: pnpm
- **Test Framework**: Vitest
- **Linter/Formatter**: oxlint / oxfmt
- **Deployment**: Docker (production runs as `openclaw:local` image)

## Essential Commands

```bash
pnpm install              # Install dependencies
pnpm build                # Build TypeScript to dist/
pnpm dev                  # Run dev server
pnpm test:fast            # Run unit tests (vitest, fast config)
pnpm test                 # Run full parallel test suite
pnpm check                # Format check + type check + lint
pnpm lint                 # Run oxlint
pnpm format               # Run oxfmt --write
```

### Docker

```bash
docker build -t openclaw:test .     # Build test image (safe)
docker build -t openclaw:local .    # Build production image (replaces running image)
```

## Git & Upstream Workflow

This is a fork. All upstream sync operations must use git — never GitHub's "Sync fork" button, as we want full control over merge conflicts and validation.

### Remotes

| Remote | URL | Purpose |
|--------|-----|---------|
| `origin` | `TrueNorthTeamsAI/personal-2nd-brain-openclaw` | Our fork |
| `upstream` | `openclaw/openclaw` | Parent repository |

If the `upstream` remote is missing, add it:

```bash
git remote add upstream https://github.com/openclaw/openclaw.git
```

### Syncing from Upstream

Use the `/sync-upstream` command to automate the full process. It will:

1. Fetch upstream changes
2. Merge into local main
3. Build a test Docker image (tagged `openclaw:test`, never `openclaw:local`)
4. Run the unit test suite inside the container
5. Push to a dated branch (`chore/sync-upstream-YYYY-MM-DD`)
6. Report results

**Production safety**: The running production container uses the `openclaw:local` Docker image. Syncing upstream never touches this image. Production is only affected when you explicitly rebuild `openclaw:local` and restart the container.

### Branch Naming

```
chore/sync-upstream-YYYY-MM-DD    # Upstream sync branches
feat/{description}                 # New features
fix/{description}                  # Bug fixes
```

## Architecture

```
src/                    # TypeScript source
  agents/               # AI agent runtime, tools, auth profiles
  auto-reply/           # Message reply pipeline
  channels/             # Channel plugins (telegram, discord, slack, etc.)
  cli/                  # CLI commands and TUI
  config/               # Configuration loading, validation, schema
  infra/                # Infrastructure (networking, heartbeat, ports)
  security/             # Security hardening, audit
dist/                   # Built output (committed for npm package)
ui/                     # Web UI (Vite/React)
apps/                   # Native apps (macOS, iOS, Android)
docs/                   # Mintlify documentation site
skills/                 # Bundled agent skills
scripts/                # Build and utility scripts
docker-compose.yml      # Container orchestration
Dockerfile              # Production Docker build
```

## Production Environment

Production runs on the same machine as development. The gateway container is:

- **Container**: `openclaw-gateway-prod`
- **Image**: `openclaw:local`
- **Compose file**: `docker-compose.yml`
- **Config volume**: Mapped from host `$OPENCLAW_CONFIG_DIR`

Always verify changes with `openclaw:test` before rebuilding `openclaw:local`.

## Known Test Failures (Environment-Specific)

These tests fail in Docker containers and are **not code bugs**:

- `browser-cli-extension.test.ts` — Requires Chrome extension assets not present in container
- `cron-protocol-conformance.test.ts` — Requires macOS Swift source files
- `bash-tools.test.ts` (notifyOnExit) — PTY spawn permission issue in containers

These should not block upstream syncs or deployments.
