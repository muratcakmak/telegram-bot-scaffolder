# Telegram Bot Scaffolder

A Claude Code plugin that scaffolds new Telegram bots on **Cloudflare Workers** with a battle-tested stack:

- **grammY** — Telegram Bot framework
- **Hono** — HTTP routing (webhook + optional API)
- **Cloudflare D1** — SQLite database
- **Cloudflare KV** — Key-value cache
- **Cloudflare R2** — Object storage (optional)
- **OpenRouter** — LLM integration (GPT, Grok, Claude, etc.)
- **TypeScript** — End-to-end type safety

## Installation

```
/plugin marketplace add muratcakmak/telegram-bot-scaffolder
/plugin install telegram-bot-scaffolder@muratcakmak-telegram-bot-scaffolder
```

## Usage

```
/telegram-bot-scaffolder:create-bot my-bot-name
```

The skill will interactively ask you which optional modules to include:

| Module | What it adds |
|--------|-------------|
| **Photo handling** | R2 storage, photo upload/download service |
| **Cron scheduling** | Hourly cron with timezone-aware per-user dispatch |
| **API routes** | Hono REST API with token auth for external dashboards |
| **Onboarding flow** | Multi-step user setup with state machine |
| **LLM vision** | Photo analysis via vision models (requires Photo handling) |
| **Conversation context** | KV-cached message history for multi-turn LLM conversations |

## What you get

A fully typed, deployable Cloudflare Worker with:

- Telegram webhook handler with secret-in-URL pattern
- LLM integration via OpenRouter (JSON mode + plain text)
- D1 database with users table and typed queries
- Utility helpers (markdown escaping, date formatting)
- Webhook setup script
- Example configs (wrangler.toml.example, .dev.vars.example)
- Proper .gitignore (excludes wrangler.toml with real IDs)

## After scaffolding

1. Copy `wrangler.toml.example` to `wrangler.toml` and fill in your Cloudflare resource IDs
2. Copy `.dev.vars.example` to `.dev.vars` and add your secrets
3. Create D1 database: `npx wrangler d1 create <bot-name>-db`
4. Run schema: `npx wrangler d1 execute <bot-name>-db --file=src/db/schema.sql`
5. Deploy: `npx wrangler deploy`
6. Set webhook: `npx tsx scripts/set-webhook.ts`
