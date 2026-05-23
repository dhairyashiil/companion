# Companion Chat App — Push Notification Delivery Design

**Date:** 2026-05-23

## Goal

Implement the companion chat app side of the Slack/Telegram push notification feature. The `/cal` backend (PR #2927) already dispatches enriched `ChatPushPayload` to `POST /api/notifications/deliver` on the chat app, and exposes subscription management API endpoints. This PR makes the chat app receive those pushes, format them, and deliver them — plus gives users a `/cal notify on|off` command to opt in and out.

## Context

The `/cal` backend sends a `ChatPushPayload` to `POST /api/notifications/deliver` with pre-resolved subscriber identifiers. The chat app is purely the mailroom: validate auth, format the message, deliver to each subscriber, report per-identifier success/failure. Subscriptions are stored in `/cal`'s Prisma DB — the chat app has no subscription state of its own.

Existing flows are unchanged: the Cal.com webhook handler (`/api/webhooks/calcom`) continues to work independently.

---

## Architecture — Option A (thin route + dedicated modules)

```
apps/chat/
├── app/api/notifications/deliver/route.ts      ← NEW: auth + body validation only
└── lib/
    ├── push-notifications/
    │   ├── formatter.ts                        ← NEW: ChatPushPayload type + buildPushCard()
    │   ├── deliver-slack.ts                    ← NEW: per-identifier Slack DM
    │   ├── deliver-telegram.ts                 ← NEW: per-identifier Telegram send
    │   └── service.ts                          ← NEW: deliverNotifications() fan-out
    ├── calcom/
    │   └── client.ts                           ← MODIFY: add 4 subscription methods
    ├── handlers/
    │   ├── slack.ts                            ← MODIFY: add "notify" subcommand
    │   └── telegram.ts                         ← MODIFY: add "notify" to TELEGRAM_COMMANDS + handler
    └── env.ts                                  ← MODIFY: validate CALCOM_DELIVERY_SECRET
```

---

## Components

### 1. `POST /api/notifications/deliver` route

**Auth:** Reads `x-cal-delivery-secret` header, compares with `CALCOM_DELIVERY_SECRET` env var using constant-time comparison (`timingSafeEqual`) — same pattern as `lib/calcom/webhooks.ts` and `lib/calcom/oauth.ts`. Returns 401 if missing or wrong. Returns 400 if body fails schema validation.

The route stays thin — auth + parse body + delegate:

```ts
export async function POST(request: Request) {
  // 1. verify secret → 401
  // 2. parse + validate body → 400
  // 3. const results = await deliverNotifications(body)
  // 4. return Response.json({ results })
}
```

**Input schema (discriminated union — `teamId` required for Slack, absent for Telegram):**
```ts
type DeliverRequest =
  | { platform: "SLACK";    subscriptions: Array<{ identifier: string; teamId: string }>; payload: ChatPushPayload }
  | { platform: "TELEGRAM"; subscriptions: Array<{ identifier: string }>;                  payload: ChatPushPayload };
```

**Output:**
```ts
{ results: Array<{ identifier: string; success: boolean; invalidIdentifier?: boolean }> }
```

Always returns 200 with the results array — errors are per-identifier, not HTTP-level (except auth/parse failure).

---

### 2. `lib/push-notifications/service.ts`

Owns the fan-out logic, keeping the route handler dumb:

```ts
export async function deliverNotifications(request: DeliverRequest): Promise<DeliverResult[]>
```

- Calls `buildPushCard(request.payload)` once to build the shared card
- Fans out to all identifiers via `Promise.allSettled()` (parallel delivery)
- Delegates per-identifier work to `deliverSlack` or `deliverTelegram`
- Returns the flat results array

`Promise.allSettled` is appropriate: the subscriber count per request is bounded by a single booking's subscriber list (typically <50), well within Slack's rate limits for bot DMs.

---

### 3. `lib/push-notifications/formatter.ts`

Defines `ChatPushPayload` locally — mirrors `/cal` PR #2927's type, no cross-repo import needed. A comment in the file links to PR #2927 so future drift is detectable.

```ts
// Mirrors ChatPushPayload from calcom/cal PR #2927 (packages/features/notifications/send-chat-push-notification.ts)
export type ChatPushPayload = { ... }
```

Exports:

```ts
export function buildPushCard(payload: ChatPushPayload): ChatElement
```

Return type is `ChatElement` (the Chat SDK's generic element type) — more portable than `ReturnType<typeof Card>`.

Renders using existing Chat SDK components (`Card`, `Fields`, `Field`, `Divider`, `Actions`, `LinkButton`) — same pattern as `lib/notifications.ts`:

**Card layout (matches approved visual):**
- `title`: event badge with emoji per type:
  - `"✅ Booking Confirmed"` / `"❌ Booking Cancelled"` / `"🔄 Booking Rescheduled"` / `"🕐 Booking Requested"` / `"🚫 Booking Rejected"`
- `subtitle`: booking title (e.g. `"Bi-Weekly Morale Talk"`)
- `Fields`:
  - `When`: `formatPushTime(start, end, timeZone)` — local helper formatting ISO strings + timezone into `"Wed Jun 3 · 4:00–4:30 PM IST"`
  - `Hosts`: `name · email` joined by `, ` — omitted if empty array
  - `Attendees`: same format — omitted if empty array
  - `Meeting`: `meetingUrl` as a link — falls back to `Location` plain text if `meetingUrl` absent but `location` present; omitted if both absent
  - `Reason`: cancellation reason — only when `notificationType === "BOOKING_CANCELLED"` and `cancellationReason` is set
- `Actions`: `LinkButton({ url: payload.data?.url ?? "https://app.cal.com/bookings", label: "View Booking" })`

---

### 4. `lib/push-notifications/deliver-slack.ts`

```ts
export async function deliverSlack(
  identifier: string,  // Slack user ID (U...)
  teamId: string,      // Slack workspace ID (T...)
  card: ChatElement,
): Promise<DeliverResult>
```

Uses the existing pattern from `app/api/webhooks/calcom/route.ts:108-114`:
1. `slackAdapter.getInstallation(teamId)` — null → `invalidIdentifier: true` (workspace uninstalled)
2. `slackAdapter.withBotToken(installation.botToken, () => bot.channel("slack:" + identifier).post(card))`
3. Catches Slack API error strings → `invalidIdentifier: true` for: `channel_not_found`, `not_in_channel`, `account_inactive`; all other errors → `success: false`

---

### 5. `lib/push-notifications/deliver-telegram.ts`

```ts
export async function deliverTelegram(
  identifier: string,  // Telegram chat ID (numeric string)
  card: ChatElement,
): Promise<DeliverResult>
```

1. `bot.channel("telegram:" + identifier).post(card)`
2. Catches errors: chat not found → `invalidIdentifier: true`; other errors → `success: false`

---

### 6. Subscribe/unsubscribe slash commands

**Slack — `/cal notify on|off`**

Added as `case "notify":` to the existing `switch(subcommand)` in `lib/handlers/slack.ts:394`:

- `on`: `getValidAccessToken` (existing pattern) → error if not linked → `registerSlackSubscription(accessToken, { identifier: userId, deviceId: teamId })` → ephemeral `"✅ You'll now receive booking notifications here."`
- `off`: `removeSlackSubscription(accessToken, { identifier: userId })` → ephemeral `"🔕 Booking push notifications turned off."`
- No arg / unknown: ephemeral usage hint: `` "Usage: `/cal notify on` or `/cal notify off`" ``

No Slack manifest changes needed — `/cal notify` is a subcommand of the existing `/cal` command.

**Telegram — `/notify on|off`**

Two code changes required (not just BotFather config):
1. Add `"notify"` to `TELEGRAM_COMMANDS` array at `lib/handlers/telegram.ts:45` — this updates the `TELEGRAM_COMMAND_RE` regex automatically since it's built from that array
2. Add `if (cmd === "notify")` handler block following the existing if-chain pattern

- `on`: linked account required → `registerTelegramSubscription(accessToken, { identifier: chatId })` → `"✅ You'll now receive booking notifications here."`
- `off`: `removeTelegramSubscription(accessToken, { identifier: chatId })` → `"🔕 Booking push notifications turned off."`

BotFather: add `/notify - Toggle booking push notifications on/off` as an operational step after deployment.

---

### 7. Cal.com client extension (`lib/calcom/client.ts`)

Four new exported functions using `fetchWithRetry` + user `accessToken` — identical pattern to existing `cancelBooking`, `rescheduleBooking` etc.:

```ts
// Endpoints confirmed from /cal PR #2927 (apps/api/v2/.../notifications-chat-subscriptions.controller.ts)
registerSlackSubscription(accessToken: string, input: { identifier: string; deviceId: string }): Promise<void>
  // POST /v2/notifications/subscriptions/slack

removeSlackSubscription(accessToken: string, input: { identifier: string }): Promise<void>
  // DELETE /v2/notifications/subscriptions/slack

registerTelegramSubscription(accessToken: string, input: { identifier: string }): Promise<void>
  // POST /v2/notifications/subscriptions/telegram

removeTelegramSubscription(accessToken: string, input: { identifier: string }): Promise<void>
  // DELETE /v2/notifications/subscriptions/telegram
```

All throw `CalcomApiError` on failure, handled by the slash command's existing `friendlyCalcomError` helper.

---

### 8. Environment variables

**New:** `CALCOM_DELIVERY_SECRET` — must match `CALCOM_CHAT_DELIVERY_SECRET` on the `/cal` side.

Validation in `lib/env.ts` matching the `REDIS_URL` pattern (line 19-23):
- Production: added to `missing` array → hard fail on startup
- Development: `console.warn` only (same as REDIS_URL behaviour)

Added to `.env.example` with description.

---

## Error Handling

| Scenario | Behavior |
|---|---|
| Wrong/missing `x-cal-delivery-secret` | 401, no body |
| Malformed / invalid request body | 400, no body |
| Slack workspace uninstalled (`getInstallation` → null) | `invalidIdentifier: true` — `/cal` cleans up subscription |
| Slack errors: `channel_not_found`, `not_in_channel`, `account_inactive` | `invalidIdentifier: true` |
| Slack transient error | `success: false` |
| Telegram chat not found | `invalidIdentifier: true` |
| Telegram transient error | `success: false` |
| `/cal notify on` — user not linked | Ephemeral: "Link your Cal.com account first with `/cal link`" |
| `/cal notify on/off` — API error | Ephemeral via existing `friendlyCalcomError` helper |

---

## Out of Scope

- No changes to the existing Cal.com webhook flow (`/api/webhooks/calcom`)
- No new Redis state (subscriptions live in `/cal` Prisma)
- Feature flag (`chat-push-notifications`) controlled on `/cal` side — chat app is always ready once deployed
- No Slack manifest changes (`/cal notify` is a subcommand of existing `/cal`)
- No rate limiting on the deliver endpoint — subscriber count per request is bounded by a single booking's list; `/cal` manages batching
