---
name: create-bot
description: Scaffold a new Telegram bot on Cloudflare Workers with grammY, D1, KV, R2, and OpenRouter. Use when starting a new Telegram bot project.
user-invocable: true
allowed-tools: Read, Write, Bash, AskUserQuestion
argument-hint: [bot-name]
---

# Telegram Bot Scaffolder

You are scaffolding a new Telegram bot project using Cloudflare Workers. Follow these steps precisely.

## Step 1: Gather Information

If `$ARGUMENTS` is provided, use it as the bot name. Otherwise, ask the user.

Ask the user for:
1. **Bot name** (kebab-case, e.g. `habit-tracker`) — this becomes the directory name and wrangler project name
2. **Bot description** (one line, e.g. "A bot that tracks daily habits and sends reminders")
3. **Which optional modules to include** (multi-select, the user can pick any combination):
   - **Photo handling** — R2 bucket for storing/serving photos
   - **Cron scheduling** — Hourly cron trigger with timezone-aware per-user dispatch
   - **API routes** — Hono REST API with token auth for external dashboards
   - **Onboarding flow** — Multi-step user setup wizard with state machine
   - **LLM vision** — Photo analysis via vision model (requires Photo handling)
   - **Conversation context** — KV-cached message history for multi-turn LLM conversations

## Step 2: Scaffold Core Files

Create all files inside a `<bot-name>/` directory in the current working directory.

### `package.json`

```json
{
  "name": "<bot-name>",
  "version": "0.1.0",
  "private": true,
  "description": "<bot-description>",
  "scripts": {
    "dev": "wrangler dev",
    "deploy": "wrangler deploy",
    "db:migrate": "wrangler d1 execute <bot-name>-db --file=src/db/schema.sql",
    "db:migrate:local": "wrangler d1 execute <bot-name>-db --local --file=src/db/schema.sql",
    "set-webhook": "tsx scripts/set-webhook.ts"
  },
  "dependencies": {
    "grammy": "^1.35.0",
    "hono": "^4.7.0"
  },
  "devDependencies": {
    "@cloudflare/workers-types": "^4.20250214.0",
    "tsx": "^4.19.0",
    "typescript": "^5.7.0",
    "wrangler": "^4.0.0"
  }
}
```

### `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "bundler",
    "lib": ["ES2022"],
    "types": ["@cloudflare/workers-types"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

### `.gitignore`

```
node_modules/
dist/
.dev.vars
.wrangler/
wrangler.toml
```

### `.dev.vars.example`

```
TELEGRAM_BOT_TOKEN=your-telegram-bot-token
OPENROUTER_API_KEY=your-openrouter-api-key
WEBHOOK_SECRET=a-random-secret-string
```

If **API routes** module is selected, also add:
```
API_TOKEN=your-api-auth-token
```

### `wrangler.toml.example`

```toml
name = "<bot-name>"
main = "src/index.ts"
compatibility_date = "2025-02-01"

[[d1_databases]]
binding = "DB"
database_name = "<bot-name>-db"
database_id = "YOUR_D1_DATABASE_ID"
```

If **Conversation context** module is selected, add:
```toml
[[kv_namespaces]]
binding = "KV"
id = "YOUR_KV_NAMESPACE_ID"
```

If **Photo handling** module is selected, add:
```toml
[[r2_buckets]]
binding = "PHOTOS"
bucket_name = "<bot-name>-photos"

[vars]
PHOTOS_PUBLIC_URL = "https://your-r2-public-url.com"
```

If **Cron scheduling** module is selected, add:
```toml
[triggers]
crons = ["0 * * * *"]
```

### `src/config.ts`

```typescript
export interface Env {
  // Bindings
  DB: D1Database;
  // Secrets
  TELEGRAM_BOT_TOKEN: string;
  OPENROUTER_API_KEY: string;
  WEBHOOK_SECRET: string;
}
```

**Conditionally add to the Env interface:**
- If **Conversation context**: `KV: KVNamespace;` in bindings
- If **Photo handling**: `PHOTOS: R2Bucket;` in bindings and `PHOTOS_PUBLIC_URL: string;` in secrets/vars
- If **API routes**: `API_TOKEN: string;` in secrets

### `src/index.ts`

```typescript
import { Hono } from "hono";
import { createBot, createWebhookHandler } from "./bot";
import type { Env } from "./config";

const app = new Hono<{ Bindings: Env }>();

app.post("/webhook/:secret", async (c) => {
  const secret = c.req.param("secret");
  if (secret !== c.env.WEBHOOK_SECRET) {
    return c.text("Unauthorized", 401);
  }
  const handler = createWebhookHandler(c.env);
  return handler(c.req.raw);
});

app.get("/", (c) => c.text("<bot-name> is running"));

export default {
  fetch: app.fetch,
} satisfies ExportedHandler<Env>;
```

**Conditionally add:**
- If **API routes**: Import and mount the API router: `import { apiRouter } from "./api/routes";` and `app.route("/api", apiRouter(c.env));` — or better, mount via `app.route("/api", apiRoutes);` with middleware that reads env from context.
- If **Cron scheduling**: Add `scheduled` export:
  ```typescript
  export default {
    fetch: app.fetch,
    scheduled: async (event: ScheduledEvent, env: Env, ctx: ExecutionContext) => {
      const { handleScheduled } = await import("./cron/scheduled");
      ctx.waitUntil(handleScheduled(env, event.scheduledTime));
    },
  } satisfies ExportedHandler<Env>;
  ```

### `src/bot.ts`

```typescript
import { Bot, webhookCallback } from "grammy";
import type { Env } from "./config";
import { handleMessage } from "./handlers/message";

export function createBot(env: Env): Bot {
  const bot = new Bot(env.TELEGRAM_BOT_TOKEN);

  bot.command("start", async (ctx) => {
    await ctx.reply(
      "Welcome! I'm <bot-name>. Send me a message to get started.",
      { parse_mode: "MarkdownV2" }
    );
  });

  bot.command("help", async (ctx) => {
    await ctx.reply("Here are the available commands:\n/start - Start the bot\n/help - Show this help message");
  });

  bot.on("message:text", async (ctx) => {
    await handleMessage(ctx, env);
  });

  return bot;
}

export function createWebhookHandler(env: Env) {
  const bot = createBot(env);
  return webhookCallback(bot, "cloudflare-mod");
}
```

**Conditionally add:**
- If **Photo handling**: Add `bot.on("message:photo", ...)` handler
- If **Onboarding flow**: Import and call onboarding handler before message handler, checking if user is in onboarding

### `src/handlers/message.ts`

```typescript
import type { Context } from "grammy";
import type { Env } from "../config";
import { callLLM } from "../services/llm";
import { getUser, upsertUser } from "../db/queries";

export async function handleMessage(ctx: Context, env: Env): Promise<void> {
  const telegramId = ctx.from?.id;
  const text = ctx.message?.text;

  if (!telegramId || !text) return;

  // Ensure user exists
  let user = await getUser(env, telegramId);
  if (!user) {
    await upsertUser(env, telegramId, ctx.from?.first_name ?? "User");
    user = await getUser(env, telegramId);
  }

  // Parse user intent via LLM
  const parsed = await callLLM<{ intent: string; data: Record<string, unknown> }>(env, [
    {
      role: "system",
      content: `You are a helpful assistant for <bot-name>. <bot-description>.
Parse the user's message and return JSON with:
- "intent": one of "general_question", "unknown"
- "data": any extracted structured data

Respond ONLY with valid JSON.`,
    },
    { role: "user", content: text },
  ]);

  // Route by intent
  switch (parsed.intent) {
    default:
      await ctx.reply("I received your message! This is where your bot logic goes.");
  }
}
```

**Conditionally add:**
- If **Conversation context**: Load context before LLM call and save after:
  ```typescript
  import { getContext, saveContext } from "../services/context";
  // Before LLM call:
  const history = await getContext(env, telegramId);
  // Include history in messages array
  // After response:
  await saveContext(env, telegramId, [...history, { role: "user", content: text }, { role: "assistant", content: response }]);
  ```

### `src/services/llm.ts`

```typescript
import type { Env } from "../config";

interface ChatMessage {
  role: "system" | "user" | "assistant";
  content: string;
}

export async function callLLM<T = Record<string, unknown>>(
  env: Env,
  messages: ChatMessage[]
): Promise<T> {
  const response = await fetch("https://openrouter.ai/api/v1/chat/completions", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${env.OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "x-ai/grok-4-fast",
      messages,
      response_format: { type: "json_object" },
    }),
  });

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`OpenRouter API error: ${response.status} ${error}`);
  }

  const data = (await response.json()) as {
    choices: Array<{ message: { content: string } }>;
  };
  const content = data.choices[0]?.message?.content;
  if (!content) throw new Error("Empty LLM response");

  return JSON.parse(content) as T;
}

export async function callLLMText(
  env: Env,
  messages: ChatMessage[]
): Promise<string> {
  const response = await fetch("https://openrouter.ai/api/v1/chat/completions", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${env.OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "x-ai/grok-4-fast",
      messages,
    }),
  });

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`OpenRouter API error: ${response.status} ${error}`);
  }

  const data = (await response.json()) as {
    choices: Array<{ message: { content: string } }>;
  };
  return data.choices[0]?.message?.content ?? "";
}
```

**Conditionally add:**
- If **LLM vision**: Add `callLLMVision` function:
  ```typescript
  export async function callLLMVision(
    env: Env,
    imageUrl: string,
    prompt: string
  ): Promise<string> {
    const response = await fetch("https://openrouter.ai/api/v1/chat/completions", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${env.OPENROUTER_API_KEY}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: "x-ai/grok-4-fast",
        messages: [
          {
            role: "user",
            content: [
              { type: "text", text: prompt },
              { type: "image_url", image_url: { url: imageUrl } },
            ],
          },
        ],
      }),
    });

    if (!response.ok) {
      const error = await response.text();
      throw new Error(`OpenRouter Vision API error: ${response.status} ${error}`);
    }

    const data = (await response.json()) as {
      choices: Array<{ message: { content: string } }>;
    };
    return data.choices[0]?.message?.content ?? "";
  }
  ```

### `src/db/schema.sql`

```sql
CREATE TABLE IF NOT EXISTS users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  telegram_id INTEGER UNIQUE NOT NULL,
  name TEXT NOT NULL DEFAULT 'User',
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX IF NOT EXISTS idx_users_telegram_id ON users(telegram_id);
```

**Conditionally add columns:**
- If **Cron scheduling**: Add `timezone TEXT NOT NULL DEFAULT 'America/New_York'` column
- If **Onboarding flow**: Add `onboarding_step INTEGER NOT NULL DEFAULT 0` column

### `src/db/queries.ts`

```typescript
import type { Env } from "../config";

export interface User {
  id: number;
  telegram_id: number;
  name: string;
  created_at: string;
  updated_at: string;
}
```

**Conditionally add to User interface:**
- If **Cron scheduling**: `timezone: string;`
- If **Onboarding flow**: `onboarding_step: number;`

```typescript
export async function getUser(env: Env, telegramId: number): Promise<User | null> {
  return env.DB.prepare("SELECT * FROM users WHERE telegram_id = ?")
    .bind(telegramId)
    .first<User>();
}

export async function upsertUser(env: Env, telegramId: number, name: string): Promise<void> {
  await env.DB.prepare(
    `INSERT INTO users (telegram_id, name) VALUES (?, ?)
     ON CONFLICT (telegram_id) DO UPDATE SET name = excluded.name, updated_at = datetime('now')`
  )
    .bind(telegramId, name)
    .run();
}

export async function getAllUsers(env: Env): Promise<User[]> {
  const result = await env.DB.prepare("SELECT * FROM users").all<User>();
  return result.results;
}
```

### `src/utils/formatting.ts`

```typescript
export function escapeMarkdown(text: string): string {
  return text.replace(/([_*\[\]()~`>#+\-=|{}.!\\])/g, "\\$1");
}

export function todayDateString(timezone = "America/New_York"): string {
  return new Date().toLocaleDateString("en-CA", { timeZone: timezone });
}

export function yesterdayDateString(timezone = "America/New_York"): string {
  const d = new Date();
  d.setDate(d.getDate() - 1);
  return d.toLocaleDateString("en-CA", { timeZone: timezone });
}
```

**Conditionally add:**
- If **Cron scheduling**:
  ```typescript
  export function getLocalHour(timezone: string, scheduledTime: number): number {
    const date = new Date(scheduledTime);
    const hourStr = new Intl.DateTimeFormat("en-US", {
      hour: "numeric",
      hour12: false,
      timeZone: timezone,
    }).format(date);
    return parseInt(hourStr, 10);
  }

  export function getLocalDayOfWeek(timezone: string, scheduledTime: number): number {
    const date = new Date(scheduledTime);
    const dayStr = new Intl.DateTimeFormat("en-US", {
      weekday: "short",
      timeZone: timezone,
    }).format(date);
    const days = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"];
    return days.indexOf(dayStr);
  }
  ```

### `scripts/set-webhook.ts`

```typescript
import { config } from "dotenv";
config({ path: ".dev.vars" });

const BOT_TOKEN = process.env.TELEGRAM_BOT_TOKEN!;
const WEBHOOK_SECRET = process.env.WEBHOOK_SECRET!;

// Update this after deploying
const WORKER_URL = "https://<bot-name>.<your-subdomain>.workers.dev";

async function setWebhook() {
  const webhookUrl = `${WORKER_URL}/webhook/${WEBHOOK_SECRET}`;
  const response = await fetch(
    `https://api.telegram.org/bot${BOT_TOKEN}/setWebhook`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        url: webhookUrl,
        allowed_updates: ["message", "callback_query"],
      }),
    }
  );
  const result = await response.json();
  console.log("Webhook set:", result);
}

setWebhook();
```

Add `dotenv` to devDependencies in package.json: `"dotenv": "^16.4.0"`

### `README.md`

Generate a README with:
- Bot name and description
- Setup instructions (create D1 database, copy config files, set secrets)
- Deployment steps
- Available commands

## Step 3: Scaffold Conditional Module Files

Only create these files if the corresponding module was selected.

### Photo handling → `src/services/photos.ts`

```typescript
import type { Env } from "../config";

export async function storePhoto(
  env: Env,
  telegramId: number,
  type: string,
  buffer: ArrayBuffer
): Promise<string> {
  const timestamp = Date.now();
  const rand = Math.random().toString(36).substring(2, 8);
  const key = `${telegramId}/${type}/${timestamp}-${rand}.jpg`;
  await env.PHOTOS.put(key, buffer, {
    httpMetadata: { contentType: "image/jpeg" },
  });
  return key;
}

export function getPhotoUrl(env: Env, key: string): string {
  return `${env.PHOTOS_PUBLIC_URL}/${key}`;
}

export async function downloadTelegramPhoto(
  botToken: string,
  fileId: string
): Promise<ArrayBuffer> {
  const fileRes = await fetch(
    `https://api.telegram.org/bot${botToken}/getFile?file_id=${fileId}`
  );
  const fileData = (await fileRes.json()) as {
    result: { file_path: string };
  };
  const downloadRes = await fetch(
    `https://api.telegram.org/file/bot${botToken}/${fileData.result.file_path}`
  );
  return downloadRes.arrayBuffer();
}
```

### Cron scheduling → `src/cron/scheduled.ts`

```typescript
import type { Env } from "../config";
import { getAllUsers } from "../db/queries";
import { getLocalHour } from "../utils/formatting";

export async function handleScheduled(env: Env, scheduledTime: number): Promise<void> {
  const users = await getAllUsers(env);

  for (const user of users) {
    const localHour = getLocalHour(user.timezone, scheduledTime);

    // Example: send a daily reminder at 9 AM local time
    if (localHour === 9) {
      // TODO: Implement your scheduled action here
      // await sendReminder(env, user);
    }
  }
}
```

### API routes → `src/api/routes.ts`

```typescript
import { Hono } from "hono";
import type { Env } from "../config";

const api = new Hono<{ Bindings: Env }>();

// Token auth middleware
api.use("*", async (c, next) => {
  const token = c.req.header("Authorization")?.replace("Bearer ", "");
  if (token !== c.env.API_TOKEN) {
    return c.json({ error: "Unauthorized" }, 401);
  }
  await next();
});

api.get("/health", (c) => c.json({ status: "ok" }));

api.get("/summary", async (c) => {
  // TODO: Implement your summary endpoint
  return c.json({ message: "Summary endpoint — implement your logic here" });
});

export { api as apiRoutes };
```

### Onboarding flow → `src/handlers/onboarding.ts`

```typescript
import type { Context } from "grammy";
import type { Env } from "../config";
import type { User } from "../db/queries";

interface OnboardingStep {
  field: string;
  prompt: string;
  parser: (input: string) => { value: unknown; error?: string } | { value?: never; error: string };
}

// Define your onboarding steps here
const ONBOARDING_STEPS: OnboardingStep[] = [
  // Example step:
  // {
  //   field: "timezone",
  //   prompt: "What timezone are you in? (e.g. America/New_York, Europe/London)",
  //   parser: (input) => {
  //     try {
  //       Intl.DateTimeFormat(undefined, { timeZone: input });
  //       return { value: input };
  //     } catch {
  //       return { error: "Invalid timezone. Please try again." };
  //     }
  //   },
  // },
];

export function isOnboarding(user: User): boolean {
  return user.onboarding_step < ONBOARDING_STEPS.length;
}

export async function handleOnboarding(
  ctx: Context,
  env: Env,
  user: User
): Promise<boolean> {
  if (!isOnboarding(user)) return false;

  const step = ONBOARDING_STEPS[user.onboarding_step];
  const text = ctx.message?.text;

  if (!text) {
    await ctx.reply(step.prompt);
    return true;
  }

  const result = step.parser(text);
  if (result.error) {
    await ctx.reply(result.error);
    return true;
  }

  // Save the field value and advance step
  await env.DB.prepare(
    `UPDATE users SET ${step.field} = ?, onboarding_step = onboarding_step + 1, updated_at = datetime('now') WHERE telegram_id = ?`
  )
    .bind(result.value as string, user.telegram_id)
    .run();

  const nextStep = user.onboarding_step + 1;
  if (nextStep < ONBOARDING_STEPS.length) {
    await ctx.reply(ONBOARDING_STEPS[nextStep].prompt);
  } else {
    await ctx.reply("Setup complete! You're all set.");
  }

  return true;
}
```

### Conversation context → `src/services/context.ts`

```typescript
import type { Env } from "../config";

interface ContextMessage {
  role: "user" | "assistant";
  content: string;
}

const MAX_CONTEXT_MESSAGES = 20;
const CONTEXT_TTL_SECONDS = 60 * 60 * 24; // 24 hours

export async function getContext(
  env: Env,
  chatId: number
): Promise<ContextMessage[]> {
  const raw = await env.KV.get(`ctx:${chatId}`);
  if (!raw) return [];
  try {
    return JSON.parse(raw) as ContextMessage[];
  } catch {
    return [];
  }
}

export async function saveContext(
  env: Env,
  chatId: number,
  messages: ContextMessage[]
): Promise<void> {
  const trimmed = messages.slice(-MAX_CONTEXT_MESSAGES);
  await env.KV.put(`ctx:${chatId}`, JSON.stringify(trimmed), {
    expirationTtl: CONTEXT_TTL_SECONDS,
  });
}
```

## Step 4: Assemble and Merge

When writing files, merge conditional additions into the core files:

1. **`config.ts`** — Add all selected bindings/secrets to the `Env` interface
2. **`index.ts`** — Add cron `scheduled` export if Cron selected; mount API routes if API selected
3. **`bot.ts`** — Add photo handler if Photo handling selected; add onboarding check if Onboarding selected
4. **`message.ts`** — Add context loading/saving if Conversation context selected
5. **`llm.ts`** — Add `callLLMVision` if LLM vision selected
6. **`schema.sql`** — Add conditional columns to users table
7. **`queries.ts`** — Add conditional fields to User interface
8. **`formatting.ts`** — Add timezone helpers if Cron selected
9. **`wrangler.toml.example`** — Add KV/R2/cron config as needed
10. **`.dev.vars.example`** — Add API_TOKEN if API routes selected
11. **`package.json`** — Add dotenv to devDependencies

Write each file as a complete, working file — do NOT leave placeholder merge comments.

## Step 5: Post-Scaffold

1. Run `npm install` in the bot directory
2. Run `npx tsc --noEmit` to verify TypeScript compiles
3. If type errors occur, fix them before proceeding
4. Print the following next steps to the user:

```
Your bot has been scaffolded! Next steps:

1. cd <bot-name>
2. Copy config files:
   cp wrangler.toml.example wrangler.toml
   cp .dev.vars.example .dev.vars
3. Create your D1 database:
   npx wrangler d1 create <bot-name>-db
4. Update wrangler.toml with your database ID
5. Run the schema migration:
   npm run db:migrate
6. Add your secrets to .dev.vars:
   - TELEGRAM_BOT_TOKEN (from @BotFather)
   - OPENROUTER_API_KEY (from openrouter.ai)
   - WEBHOOK_SECRET (any random string)
7. Deploy:
   npm run deploy
8. Set your webhook:
   Update WORKER_URL in scripts/set-webhook.ts, then:
   npm run set-webhook
```
