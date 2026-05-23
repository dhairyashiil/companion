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
├── app/api/notifications/deliver/route.ts      ← NEW: auth + routing
└── lib/
    ├── push-notifications/
    │   ├── formatter.ts                        ← NEW: buildPushCard(payload)
    │   ├── deliver-slack.ts                    ← NEW: per-identifier Slack DM
    │   └── deliver-telegram.ts                 ← NEW: per-identifier Telegram send
    ├── calcom/
    │   └── client.ts                           ← MODIFY: add subscription methods
    ├── handlers/
    │   ├── slack.ts                            ← MODIFY: add "notify" subcommand
    │   └── telegram.ts                         ← MODIFY: add /notify command
    └── env.ts                                  ← MODIFY: validate CALCOM_DELIVERY_SECRET
```

---

## Components

### 1. `POST /api/notifications/deliver` route

**Auth:** Reads `x-cal-delivery-secret` header, compares with `CALCOM_DELIVERY_SECRET` env var using constant-time comparison (`timingSafeEqual`). Returns 401 if missing or wrong. Returns 400 if body fails schema validation.

**Input schema:**
```ts
type DeliverRequest = {
  platform: "SLACK" | "TELEGRAM";
  subscriptions: Array<{ identifier: string; teamId?: string }>;
  payload: ChatPushPayload;
};
```

**Output:**
```ts
{ results: Array<{ identifier: string; success: boolean; invalidIdentifier?: boolean }> }
```

**Routing:** Delegates to `deliverSlack` or `deliverTelegram` per identifier. Collects results. Always returns 200 with the results array — errors are per-identifier, not HTTP-level (except auth failure).

---

### 2. `lib/push-notifications/formatter.ts`

Defines the `ChatPushPayload` type (mirrors `/cal`'s type — no cross-repo import needed) and exports a single function:

```ts
export function buildPushCard(payload: ChatPushPayload): ReturnType<typeof Card>
```

Renders using the existing Chat SDK components (`Card`, `Fields`, `Field`, `Divider`, `Actions`, `LinkButton`) — same pattern as `lib/notifications.ts`. The Card SDK handles both Slack Block Kit and Telegram formatting transparently.

**Card layout (matches approved visual):**
- `title`: event badge — e.g. `"✅ Booking Confirmed"`, `"❌ Booking Cancelled"`, `"🔄 Booking Rescheduled"`, `"🕐 Booking Requested"`, `"🚫 Booking Rejected"`
- `subtitle`: booking title (e.g. `"Bi-Weekly Morale Talk"`)
- `Fields`:
  - `When`: formatted time range with timezone using a local `formatPushTime(start, end, timeZone)` helper
  - `Hosts`: comma-joined `name · email` — omitted if array is empty
  - `Attendees`: same format — omitted if array is empty
  - `Meeting`: meeting URL as a link — omitted if absent; falls back to `Location` field if `meetingUrl` is absent but `location` is present
  - `Reason`: cancellation reason — only present when `notificationType === "BOOKING_CANCELLED"` and `cancellationReason` is set
- `Actions`: single `LinkButton({ url: payload.data.url, label: "View Booking" })`

---

### 3. `lib/push-notifications/deliver-slack.ts`

```ts
export async function deliverSlack(
  identifier: string,   // Slack user ID (U...)
  teamId: string,       // Slack workspace ID (T...)
  card: Card,
): Promise<{ success: boolean; invalidIdentifier?: boolean }>
```

Uses the existing pattern from `app/api/webhooks/calcom/route.ts`:
1. `slackAdapter.getInstallation(teamId)` — returns null if the workspace uninstalled the app → `invalidIdentifier: true`
2. `slackAdapter.withBotToken(installation.botToken, () => bot.channel("slack:" + identifier).post(card))`
3. Catches Slack API errors: channel not found / user not found → `invalidIdentifier: true`; other errors → `success: false`

---

### 4. `lib/push-notifications/deliver-telegram.ts`

```ts
export async function deliverTelegram(
  identifier: string,   // Telegram chat ID (numeric string)
  card: Card,
): Promise<{ success: boolean; invalidIdentifier?: boolean }>
```

Simpler — no workspace lookup needed:
1. `bot.channel("telegram:" + identifier).post(card)`
2. Catches errors: chat not found → `invalidIdentifier: true`; other errors → `success: false`

---

### 5. Subscribe/unsubscribe slash commands

**Slack — `/cal notify on|off`**

Added to the existing `switch(subcommand)` in `lib/handlers/slack.ts`:

- `on`: Requires linked Cal.com account (existing `getValidAccessToken` pattern). Calls `registerSlackSubscription({ identifier: userId, deviceId: teamId })`. Replies ephemeral: `"✅ You'll now receive booking notifications here."`
- `off`: Calls `removeSlackSubscription({ identifier: userId })`. Replies ephemeral: `"🔕 Booking push notifications turned off."`
- Unknown arg: replies ephemeral with usage hint: `` "`/cal notify on` or `/cal notify off`" ``

**Telegram — `/notify on|off`**

Added to `lib/handlers/telegram.ts` following the existing command handler pattern:

- `on`: Requires linked account. Calls `registerTelegramSubscription({ identifier: chatId })`. Replies: `"✅ You'll now receive booking notifications here."`
- `off`: Calls `removeTelegramSubscription({ identifier: chatId })`. Replies: `"🔕 Booking push notifications turned off."`

---

### 6. Cal.com client extension (`lib/calcom/client.ts`)

Four new functions added alongside existing ones, all using `fetchWithRetry` and authenticated with the user's access token:

```ts
registerSlackSubscription(accessToken, input: { identifier: string; deviceId: string }): Promise<void>
removeSlackSubscription(accessToken, input: { identifier: string }): Promise<void>
registerTelegramSubscription(accessToken, input: { identifier: string }): Promise<void>
removeTelegramSubscription(accessToken, input: { identifier: string }): Promise<void>
```

Endpoints: `POST /v2/notifications/subscriptions/slack`, `DELETE /v2/notifications/subscriptions/slack`, and Telegram equivalents. All return 200/201 on success; throw `CalcomApiError` on failure (handled by the calling slash command handler).

---

### 7. Environment variables

**New:** `CALCOM_DELIVERY_SECRET` — must match `CALCOM_CHAT_DELIVERY_SECRET` on the `/cal` side.

Added to `lib/env.ts` validation (warn if missing in development, hard fail in production). Added to `.env.example` with description.

---

## Error Handling

| Scenario | Behavior |
|---|---|
| Wrong/missing delivery secret | 401, no body |
| Malformed request body | 400, no body |
| Slack workspace uninstalled | `invalidIdentifier: true` — `/cal` will clean up the subscription |
| Telegram chat not found | `invalidIdentifier: true` |
| Transient delivery error | `success: false`, `/cal` retries on next event |
| `/cal notify on` — user not linked | Ephemeral error: "Link your Cal.com account first with `/cal link`" |
| `/cal notify on` — API error | Ephemeral error with friendly message via existing `friendlyCalcomError` helper |

---

## Out of Scope

- No changes to the existing Cal.com webhook flow (`/api/webhooks/calcom`)
- No new Redis state (subscriptions live in `/cal` Prisma)
- Feature flag (`chat-push-notifications`) is controlled on `/cal` side — chat app is always ready to receive once deployed
- No Slack manifest changes (no new slash commands — `/cal notify` is a subcommand of existing `/cal`)
- Telegram: `/notify` is a new bot command — update BotFather description but no code change needed
