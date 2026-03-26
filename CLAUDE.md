# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

LINE Harness is an open-source LINE Official Account CRM / marketing automation platform. It runs on Cloudflare Workers (free tier) with D1 (SQLite) as the database. Japanese-language product targeting Japanese market.

## Commands

```bash
# Install
pnpm install

# Development
pnpm dev:worker          # Hono API server → http://localhost:8787
pnpm dev:web             # Next.js admin UI → http://localhost:3001

# Database
pnpm db:migrate          # Apply schema to remote D1
pnpm db:migrate:local    # Apply schema to local D1

# Build
pnpm build               # Build all packages

# Deploy
pnpm deploy:worker       # Deploy worker to Cloudflare
pnpm deploy:web          # Build Next.js static export

# SDK tests (vitest)
cd packages/sdk && pnpm test          # Run all tests
cd packages/sdk && pnpm test:watch    # Watch mode
cd packages/sdk && npx vitest run tests/resources/friends.test.ts  # Single test file
```

## Architecture

**Monorepo** (pnpm workspaces): 3 apps + 4 packages. TypeScript throughout, strict mode.

### Apps

- **apps/worker** — Cloudflare Workers API (Hono 4.6). Entry: `src/index.ts`. Routes in `src/routes/` (24 modules, 100+ endpoints). Services in `src/services/` (step delivery, broadcast processing, ban monitoring). Auth middleware uses Bearer token (`API_KEY`), skipped for `/webhook` and public endpoints. Cron runs every 5 min (`*/5 * * * *`) processing step deliveries, broadcasts, reminders, and health checks across all active LINE accounts.

- **apps/web** — Next.js 15 admin dashboard (React 19, App Router, static export). Tailwind CSS 4. Uses `@/lib/api` for API calls. Multi-account support via `useAccount()` context. Has Claude Code integration buttons.

- **apps/liff** — Vite SPA for LINE Front-end Framework (forms + calendar booking, embedded in LINE chat). Dev port 3002.

### Packages

- **packages/db** — D1 schema (`schema.sql`, 50+ tables) and query helper functions. Each module (friends.ts, scenarios.ts, etc.) exports typed query helpers.

- **packages/line-sdk** — LINE Messaging API wrapper. `LineClient` class, message builders (text, image, flex, video, carousel, quick reply), webhook signature verification (`verifySignature`).

- **packages/shared** — Shared TypeScript types (80+ interfaces in `types.ts`) and JST time utilities (`jstNow`, `toJstString`, `isTimeBefore`).

- **packages/sdk** — Published SDK (`@line-harness/sdk`) for external integrations / AI agents. Built with tsup (ESM + CJS). Tests use vitest.

### Key Patterns

- **Multi-account**: `line_accounts` table stores per-account credentials. Scheduled handler iterates all active accounts. Webhook signature verification routes to correct account.
- **Env bindings** (worker): `DB` (D1Database), `LINE_CHANNEL_SECRET`, `LINE_CHANNEL_ACCESS_TOKEN`, `API_KEY`, `LIFF_URL`, `LINE_LOGIN_CHANNEL_ID`, `LINE_LOGIN_CHANNEL_SECRET`, `WORKER_URL`.
- **JSX**: Worker uses Hono JSX (`jsxImportSource: "hono/jsx"`), not React.
- **Template variables**: `{{name}}`, `{{uid}}`, `{{auth_url:CHANNEL_ID}}` for personalized messages.

### CI/CD

GitHub Actions (`.github/workflows/deploy-worker.yml`): triggers on push to `main` affecting `apps/worker/`, `packages/db/`, `packages/shared/`, `packages/line-sdk/`. Builds shared packages first, then deploys worker via wrangler.
