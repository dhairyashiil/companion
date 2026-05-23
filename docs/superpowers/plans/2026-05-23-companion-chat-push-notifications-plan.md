# Companion Chat Push Notifications Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `POST /api/notifications/deliver` endpoint and `/cal notify on|off` slash commands to the companion chat app so it can receive booking push notifications dispatched by the `/cal` backend (PR #2927).

**Architecture:** The deliver route stays thin (auth + parse + delegate). All fan-out logic lives in `lib/push-notifications/service.ts`. Per-platform delivery and message formatting are isolated in dedicated modules. Subscription management adds four functions to the existing Cal.com API client.

**Tech Stack:** Next.js 15 App Router, TypeScript, `chat` SDK (`Card`, `Fields`, `Field`, `Divider`, `Actions`, `LinkButton`), Node.js `crypto.timingSafeEqual`, `Promise.allSettled`.

---

## File Map

**Create:**
- `apps/chat/app/api/notifications/deliver/route.ts` — auth + body parse + delegate
- `apps/chat/lib/push-notifications/formatter.ts` — `ChatPushPayload` type + `buildPushCard()`
- `apps/chat/lib/push-notifications/deliver-slack.ts` — per-identifier Slack DM
- `apps/chat/lib/push-notifications/deliver-telegram.ts` — per-identifier Telegram DM
- `apps/chat/lib/push-notifications/service.ts` — `deliverNotifications()` fan-out

**Modify:**
- `apps/chat/lib/calcom/client.ts` — add 4 subscription methods
- `apps/chat/lib/handlers/slack.ts` — add `case "notify":` to `/cal` switch
- `apps/chat/lib/handlers/telegram.ts` — add `"notify"` to `TELEGRAM_COMMANDS` + handler
- `apps/chat/lib/env.ts` — add `CALCOM_DELIVERY_SECRET` validation
- `apps/chat/.env.example` — document new env var

---

### Task 1: `ChatPushPayload` type + `buildPushCard()` formatter

**Files:**
- Create: `apps/chat/lib/push-notifications/formatter.ts`

- [ ] **Step 1: Create the formatter**

```ts
// apps/chat/lib/push-notifications/formatter.ts
import { Actions, Card, Divider, Field, Fields, LinkButton } from "chat";
import type { ChatElement } from "chat";

// Mirrors ChatPushPayload from calcom/cal PR #2927
// packages/features/notifications/send-chat-push-notification.ts
export type ChatPushPayload = {
  title: string;
  body: string;
  data?: Record<string, string>;
  notificationType:
    | "BOOKING_CONFIRMED"
    | "BOOKING_CANCELLED"
    | "BOOKING_RESCHEDULED"
    | "BOOKING_REQUESTED"
    | "BOOKING_REJECTED";
  hosts: Array<{ name: string; email: string }>;
  attendees: Array<{ name: string; email: string }>;
  start: string;
  end: string;
  timeZone: string;
  location?: string;
  meetingUrl?: string;
  cancellationReason?: string;
};

const NOTIFICATION_BADGES: Record<ChatPushPayload["notificationType"], string> = {
  BOOKING_CONFIRMED: "✅ Booking Confirmed",
  BOOKING_CANCELLED: "❌ Booking Cancelled",
  BOOKING_RESCHEDULED: "🔄 Booking Rescheduled",
  BOOKING_REQUESTED: "🕐 Booking Requested",
  BOOKING_REJECTED: "🚫 Booking Rejected",
};

function formatPushTime(start: string, end: string, timeZone: string): string {
  const startDate = new Date(start);
  const endDate = new Date(end);
  const dateFmt = new Intl.DateTimeFormat("en-US", {
    timeZone,
    weekday: "short",
    month: "short",
    day: "numeric",
  });
  const timeFmt = new Intl.DateTimeFormat("en-US", {
    timeZone,
    hour: "numeric",
    minute: "2-digit",
    hour12: true,
    timeZoneName: "short",
  });
  const startTimeFmt = new Intl.DateTimeFormat("en-US", {
    timeZone,
    hour: "numeric",
    minute: "2-digit",
    hour12: true,
  });
  return `${dateFmt.format(startDate)} · ${startTimeFmt.format(startDate)}–${timeFmt.format(endDate)}`;
}

function formatPeople(people: Array<{ name: string; email: string }>): string {
  return people.map((p) => `${p.name} · ${p.email}`).join(", ");
}

export function buildPushCard(payload: ChatPushPayload): ChatElement {
  const badge = NOTIFICATION_BADGES[payload.notificationType];
  const when = formatPushTime(payload.start, payload.end, payload.timeZone);

  const meetingField = payload.meetingUrl
    ? [Field({ label: "Meeting", value: `[Join](${payload.meetingUrl})` })]
    : payload.location
      ? [Field({ label: "Location", value: payload.location })]
      : [];

  const reasonField =
    payload.notificationType === "BOOKING_CANCELLED" && payload.cancellationReason
      ? [Field({ label: "Reason", value: payload.cancellationReason })]
      : [];

  return Card({
    title: badge,
    subtitle: payload.title,
    children: [
      Fields([
        Field({ label: "When", value: when }),
        ...(payload.hosts.length > 0
          ? [Field({ label: "Hosts", value: formatPeople(payload.hosts) })]
          : []),
        ...(payload.attendees.length > 0
          ? [Field({ label: "Attendees", value: formatPeople(payload.attendees) })]
          : []),
        ...meetingField,
        ...reasonField,
      ]),
      Divider(),
      Actions([
        LinkButton({
          url: payload.data?.url ?? "https://app.cal.com/bookings",
          label: "View Booking",
        }),
      ]),
    ],
  });
}
```

- [ ] **Step 2: Typecheck**

```bash
cd apps/chat && bun run typecheck
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add apps/chat/lib/push-notifications/formatter.ts
git commit -m "feat(chat): add ChatPushPayload type and buildPushCard formatter"
```

---

### Task 2: `DeliverResult` shared type + `deliver-slack.ts`

**Files:**
- Create: `apps/chat/lib/push-notifications/deliver-slack.ts`

- [ ] **Step 1: Create deliver-slack**

The `slackAdapter` import mirrors how `app/api/webhooks/calcom/route.ts` uses it. Slack API errors that mean the identifier is permanently invalid return `invalidIdentifier: true` — the `/cal` backend uses this to clean up the subscription.

```ts
// apps/chat/lib/push-notifications/deliver-slack.ts
import { slackAdapter } from "@/lib/slack-adapter";
import { bot } from "@/lib/bot";
import type { ChatElement } from "chat";

export type DeliverResult = {
  identifier: string;
  success: boolean;
  invalidIdentifier?: boolean;
};

const INVALID_SLACK_ERROR_CODES = new Set([
  "channel_not_found",
  "not_in_channel",
  "account_inactive",
]);

export async function deliverSlack(
  identifier: string,
  teamId: string,
  card: ChatElement
): Promise<DeliverResult> {
  const installation = await slackAdapter.getInstallation(teamId);
  if (!installation) {
    return { identifier, success: false, invalidIdentifier: true };
  }

  try {
    await slackAdapter.withBotToken(installation.botToken, async () => {
      await bot.channel(`slack:${identifier}`).post(card);
    });
    return { identifier, success: true };
  } catch (err) {
    const code = (err as { code?: string })?.code ?? String(err);
    if (INVALID_SLACK_ERROR_CODES.has(code)) {
      return { identifier, success: false, invalidIdentifier: true };
    }
    return { identifier, success: false };
  }
}
```

- [ ] **Step 2: Typecheck**

```bash
cd apps/chat && bun run typecheck
```

Expected: no errors (the `bot` and `slackAdapter` imports resolve through the `@/` alias).

- [ ] **Step 3: Commit**

```bash
git add apps/chat/lib/push-notifications/deliver-slack.ts
git commit -m "feat(chat): add deliverSlack per-identifier DM sender"
```

---

### Task 3: `deliver-telegram.ts`

**Files:**
- Create: `apps/chat/lib/push-notifications/deliver-telegram.ts`

- [ ] **Step 1: Create deliver-telegram**

Telegram chat IDs are numeric strings. "Chat not found" errors mean the user has blocked the bot or never started a conversation — the identifier is permanently invalid.

```ts
// apps/chat/lib/push-notifications/deliver-telegram.ts
import { bot } from "@/lib/bot";
import type { ChatElement } from "chat";
import type { DeliverResult } from "./deliver-slack";

const TELEGRAM_NOT_FOUND_PHRASES = ["chat not found", "user not found", "bot was blocked by the user"];

export async function deliverTelegram(
  identifier: string,
  card: ChatElement
): Promise<DeliverResult> {
  try {
    await bot.channel(`telegram:${identifier}`).post(card);
    return { identifier, success: true };
  } catch (err) {
    const msg = String(err).toLowerCase();
    const isInvalid = TELEGRAM_NOT_FOUND_PHRASES.some((phrase) => msg.includes(phrase));
    if (isInvalid) {
      return { identifier, success: false, invalidIdentifier: true };
    }
    return { identifier, success: false };
  }
}
```

- [ ] **Step 2: Typecheck**

```bash
cd apps/chat && bun run typecheck
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add apps/chat/lib/push-notifications/deliver-telegram.ts
git commit -m "feat(chat): add deliverTelegram per-identifier DM sender"
```

---

### Task 4: `service.ts` — fan-out logic

**Files:**
- Create: `apps/chat/lib/push-notifications/service.ts`

- [ ] **Step 1: Create the service**

`Promise.allSettled` ensures one failing delivery never blocks others. Subscriber count per request is bounded by a single booking's subscriber list (typically <50), so the parallel approach is safe and has no rate-limit concerns.

```ts
// apps/chat/lib/push-notifications/service.ts
import { buildPushCard, type ChatPushPayload } from "./formatter";
import { deliverSlack, type DeliverResult } from "./deliver-slack";
import { deliverTelegram } from "./deliver-telegram";

export type DeliverRequest =
  | {
      platform: "SLACK";
      subscriptions: Array<{ identifier: string; teamId: string }>;
      payload: ChatPushPayload;
    }
  | {
      platform: "TELEGRAM";
      subscriptions: Array<{ identifier: string }>;
      payload: ChatPushPayload;
    };

export async function deliverNotifications(request: DeliverRequest): Promise<DeliverResult[]> {
  const card = buildPushCard(request.payload);

  const settled = await Promise.allSettled(
    request.subscriptions.map((sub) => {
      if (request.platform === "SLACK") {
        const slackSub = sub as { identifier: string; teamId: string };
        return deliverSlack(slackSub.identifier, slackSub.teamId, card);
      }
      return deliverTelegram(sub.identifier, card);
    })
  );

  return settled.map((result, i) => {
    if (result.status === "fulfilled") return result.value;
    return {
      identifier: request.subscriptions[i].identifier,
      success: false,
    };
  });
}
```

- [ ] **Step 2: Typecheck**

```bash
cd apps/chat && bun run typecheck
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add apps/chat/lib/push-notifications/service.ts
git commit -m "feat(chat): add deliverNotifications fan-out service"
```

---

### Task 5: `POST /api/notifications/deliver` route

**Files:**
- Create: `apps/chat/app/api/notifications/deliver/route.ts`

- [ ] **Step 1: Create the route**

`timingSafeEqual` prevents timing attacks when comparing secrets — same pattern as `lib/calcom/webhooks.ts`. The route stays thin: auth → parse → delegate → respond.

```ts
// apps/chat/app/api/notifications/deliver/route.ts
import crypto from "node:crypto";
import { deliverNotifications, type DeliverRequest } from "@/lib/push-notifications/service";

function verifyDeliverySecret(header: string | null): boolean {
  const secret = process.env.CALCOM_DELIVERY_SECRET;
  if (!secret || !header) return false;
  try {
    return crypto.timingSafeEqual(Buffer.from(header), Buffer.from(secret));
  } catch {
    return false;
  }
}

function parseDeliverRequest(body: unknown): DeliverRequest | null {
  if (typeof body !== "object" || body === null) return null;
  const b = body as Record<string, unknown>;
  if (b.platform !== "SLACK" && b.platform !== "TELEGRAM") return null;
  if (!Array.isArray(b.subscriptions) || b.subscriptions.length === 0) return null;
  if (typeof b.payload !== "object" || b.payload === null) return null;
  return body as DeliverRequest;
}

export async function POST(request: Request) {
  const secret = request.headers.get("x-cal-delivery-secret");
  if (!verifyDeliverySecret(secret)) {
    return new Response(null, { status: 401 });
  }

  let body: unknown;
  try {
    body = await request.json();
  } catch {
    return new Response(null, { status: 400 });
  }

  const parsed = parseDeliverRequest(body);
  if (!parsed) {
    return new Response(null, { status: 400 });
  }

  const results = await deliverNotifications(parsed);
  return Response.json({ results });
}
```

- [ ] **Step 2: Typecheck**

```bash
cd apps/chat && bun run typecheck
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add apps/chat/app/api/notifications/deliver/route.ts
git commit -m "feat(chat): add POST /api/notifications/deliver route"
```

---

### Task 6: Env var validation + `.env.example`

**Files:**
- Modify: `apps/chat/lib/env.ts`
- Modify: `apps/chat/.env.example`

- [ ] **Step 1: Add `CALCOM_DELIVERY_SECRET` validation to `lib/env.ts`**

Read current content first: `apps/chat/lib/env.ts`

Add after the `REDIS_URL` block (after line 23), before the `missing.length > 0` check:

```ts
  if (process.env.NODE_ENV === "production" && !process.env.CALCOM_DELIVERY_SECRET) {
    throw new Error(
      "CALCOM_DELIVERY_SECRET is required in production. Set it to the same value as CALCOM_CHAT_DELIVERY_SECRET on the /cal backend."
    );
  } else if (!process.env.CALCOM_DELIVERY_SECRET) {
    console.warn("CALCOM_DELIVERY_SECRET not set — POST /api/notifications/deliver will reject all requests.");
  }
```

- [ ] **Step 2: Add to `.env.example`**

Append to the `# ─── Cal.com ─────` section, after `CALCOM_APP_URL`:

```
# Shared secret for the push-notification delivery endpoint (POST /api/notifications/deliver).
# Must match CALCOM_CHAT_DELIVERY_SECRET on the /cal backend.
# Generate with: openssl rand -hex 32
CALCOM_DELIVERY_SECRET=your-delivery-secret
```

- [ ] **Step 3: Typecheck**

```bash
cd apps/chat && bun run typecheck
```

Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add apps/chat/lib/env.ts apps/chat/.env.example
git commit -m "feat(chat): add CALCOM_DELIVERY_SECRET env var validation"
```

---

### Task 7: Cal.com client — subscription methods

**Files:**
- Modify: `apps/chat/lib/calcom/client.ts`

- [ ] **Step 1: Add four subscription functions**

Read current content of `apps/chat/lib/calcom/client.ts` then append the following at the end of the file. The endpoints are confirmed from `/cal` PR #2927 (`apps/api/v2/.../notifications-chat-subscriptions.controller.ts`).

```ts
// ─── Chat push notification subscriptions ────────────────────────────────────

export async function registerSlackSubscription(
  accessToken: string,
  input: { identifier: string; deviceId: string }
): Promise<void> {
  await calcomFetch<void>("/v2/notifications/subscriptions/slack", accessToken, {
    method: "POST",
    body: JSON.stringify(input),
  }, API_VERSION, 0);
}

export async function removeSlackSubscription(
  accessToken: string,
  input: { identifier: string }
): Promise<void> {
  await calcomFetch<void>("/v2/notifications/subscriptions/slack", accessToken, {
    method: "DELETE",
    body: JSON.stringify(input),
  }, API_VERSION, 0);
}

export async function registerTelegramSubscription(
  accessToken: string,
  input: { identifier: string }
): Promise<void> {
  await calcomFetch<void>("/v2/notifications/subscriptions/telegram", accessToken, {
    method: "POST",
    body: JSON.stringify(input),
  }, API_VERSION, 0);
}

export async function removeTelegramSubscription(
  accessToken: string,
  input: { identifier: string }
): Promise<void> {
  await calcomFetch<void>("/v2/notifications/subscriptions/telegram", accessToken, {
    method: "DELETE",
    body: JSON.stringify(input),
  }, API_VERSION, 0);
}
```

- [ ] **Step 2: Typecheck**

```bash
cd apps/chat && bun run typecheck
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add apps/chat/lib/calcom/client.ts
git commit -m "feat(chat): add Slack/Telegram subscription methods to Cal.com client"
```

---

### Task 8: Slack `/cal notify on|off` subcommand

**Files:**
- Modify: `apps/chat/lib/handlers/slack.ts`

- [ ] **Step 1: Import the new subscription functions**

In `lib/handlers/slack.ts`, find the existing import block from `"../calcom/client"` (lines ~24–34):

```ts
import {
  CalcomApiError,
  cancelBooking,
  chargeCredits,
  createBooking,
  createBookingPublic,
  getAvailableSlotsPublic,
  getBookings,
  getEventTypesByUsername,
  getSchedules,
  rescheduleBooking,
} from "../calcom/client";
```

Add `registerSlackSubscription` and `removeSlackSubscription` to this import:

```ts
import {
  CalcomApiError,
  cancelBooking,
  chargeCredits,
  createBooking,
  createBookingPublic,
  getAvailableSlotsPublic,
  getBookings,
  getEventTypesByUsername,
  getSchedules,
  registerSlackSubscription,
  removeSlackSubscription,
  rescheduleBooking,
} from "../calcom/client";
```

- [ ] **Step 2: Add `case "notify":` to the `/cal` switch**

Find the `switch (subcommand)` block starting at line ~394. Locate the `case "help":` block (lines ~401–403). Insert `case "notify":` **before** `case "help":`:

```ts
          case "notify": {
            const notifyArg = args[1]?.toLowerCase();
            if (notifyArg !== "on" && notifyArg !== "off") {
              await event.channel.postEphemeral(
                event.user,
                "Usage: `/cal notify on` or `/cal notify off`",
                { fallbackToDM: true }
              );
              break;
            }
            const notifyToken = await getValidAccessToken(teamId, userId);
            if (!notifyToken) {
              await event.channel.postEphemeral(
                event.user,
                oauthLinkMessage("slack", teamId, userId),
                { fallbackToDM: true }
              );
              break;
            }
            if (notifyArg === "on") {
              await registerSlackSubscription(notifyToken, {
                identifier: userId,
                deviceId: teamId,
              });
              await event.channel.postEphemeral(
                event.user,
                "✅ You'll now receive booking notifications here.",
                { fallbackToDM: true }
              );
            } else {
              await removeSlackSubscription(notifyToken, { identifier: userId });
              await event.channel.postEphemeral(
                event.user,
                "🔕 Booking push notifications turned off.",
                { fallbackToDM: true }
              );
            }
            break;
          }
```

- [ ] **Step 3: Typecheck**

```bash
cd apps/chat && bun run typecheck
```

Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add apps/chat/lib/handlers/slack.ts
git commit -m "feat(chat): add /cal notify on|off Slack subcommand"
```

---

### Task 9: Telegram `/notify on|off` command

**Files:**
- Modify: `apps/chat/lib/handlers/telegram.ts`

- [ ] **Step 1: Import subscription functions**

Find the existing import from `"../calcom/client"` in `lib/handlers/telegram.ts`. It currently imports various functions. Add `registerTelegramSubscription` and `removeTelegramSubscription` to it.

Look for the `client` import block (it imports functions like `getBookings`, `getEventTypesByUsername`, etc.) and extend it:

```ts
import {
  // ... existing imports ...
  registerTelegramSubscription,
  removeTelegramSubscription,
} from "../calcom/client";
```

- [ ] **Step 2: Add `"notify"` to `TELEGRAM_COMMANDS`**

Find `TELEGRAM_COMMANDS` at line 45. Add `"notify"` to the array:

```ts
export const TELEGRAM_COMMANDS = [
  "start",
  "help",
  "link",
  "unlink",
  "notify",
  "bookings",
  "availability",
  "profile",
  "eventtypes",
  "schedules",
  "book",
  "cancel",
  "reschedule",
];
```

This automatically updates `TELEGRAM_COMMAND_RE` (built from the array on line 60) — no regex change needed.

- [ ] **Step 3: Add the `notify` handler**

Find the `if (cmd === "unlink")` block in `handleTelegramCommand`. Add the `notify` handler immediately after it, before the `if (cmd === "bookings")` block:

```ts
      if (cmd === "notify") {
        const notifyArg = rest.split(/\s+/)[0]?.toLowerCase();
        if (notifyArg !== "on" && notifyArg !== "off") {
          await postPrivately(
            thread,
            message,
            "Usage: `/notify on` or `/notify off`",
            isGroup
          );
          return;
        }
        const auth = await requireAuth();
        if (!auth) return;
        if (notifyArg === "on") {
          await registerTelegramSubscription(auth.accessToken, {
            identifier: ctx.userId,
          });
          await postPrivately(
            thread,
            message,
            "✅ You'll now receive booking notifications here.",
            isGroup
          );
        } else {
          await removeTelegramSubscription(auth.accessToken, {
            identifier: ctx.userId,
          });
          await postPrivately(
            thread,
            message,
            "🔕 Booking push notifications turned off.",
            isGroup
          );
        }
        return;
      }
```

- [ ] **Step 4: Typecheck**

```bash
cd apps/chat && bun run typecheck
```

Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add apps/chat/lib/handlers/telegram.ts
git commit -m "feat(chat): add /notify on|off Telegram command"
```

> **Post-deployment:** Add `/notify - Toggle booking push notifications on/off` in BotFather (Settings → Edit Commands).

---

### Task 10: End-to-end smoke test (manual)

No automated test runner exists in `apps/chat`. Verify correctness with these manual checks:

- [ ] **Step 1: Set env var and start dev server**

```bash
# In apps/chat/.env.local
CALCOM_DELIVERY_SECRET=test-secret-local

cd apps/chat && bun run dev
```

- [ ] **Step 2: Test auth rejection**

```bash
curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:3000/api/notifications/deliver \
  -H "Content-Type: application/json" \
  -d '{"platform":"SLACK","subscriptions":[{"identifier":"U123","teamId":"T456"}],"payload":{}}'
# Expected: 401
```

- [ ] **Step 3: Test body validation rejection**

```bash
curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:3000/api/notifications/deliver \
  -H "Content-Type: application/json" \
  -H "x-cal-delivery-secret: test-secret-local" \
  -d '{"platform":"INVALID"}'
# Expected: 400
```

- [ ] **Step 4: Test valid delivery (Telegram)**

```bash
curl -s -X POST http://localhost:3000/api/notifications/deliver \
  -H "Content-Type: application/json" \
  -H "x-cal-delivery-secret: test-secret-local" \
  -d '{
    "platform": "TELEGRAM",
    "subscriptions": [{"identifier": "YOUR_TELEGRAM_CHAT_ID"}],
    "payload": {
      "title": "Smoke test",
      "body": "test",
      "notificationType": "BOOKING_CONFIRMED",
      "hosts": [{"name": "Host", "email": "host@cal.com"}],
      "attendees": [{"name": "You", "email": "you@cal.com"}],
      "start": "2026-06-03T10:00:00.000Z",
      "end": "2026-06-03T10:30:00.000Z",
      "timeZone": "America/New_York"
    }
  }'
# Expected: {"results":[{"identifier":"YOUR_TELEGRAM_CHAT_ID","success":true}]}
```

- [ ] **Step 5: Final typecheck**

```bash
cd apps/chat && bun run typecheck
```

Expected: no errors.

- [ ] **Step 6: Commit**

No code changes — this task is verification only.

---

## Checklist — Spec Coverage

| Spec requirement | Task |
|---|---|
| `POST /api/notifications/deliver` with secret auth | Task 5 |
| `timingSafeEqual` constant-time comparison | Task 5 |
| Body validation (discriminated union) | Task 5 |
| `deliverNotifications()` fan-out via `Promise.allSettled` | Task 4 |
| `ChatPushPayload` type + `buildPushCard()` | Task 1 |
| Card fields: badge, subtitle, When, Hosts, Attendees, Meeting, Reason | Task 1 |
| `deliverSlack` — `getInstallation` + `withBotToken` + error codes | Task 2 |
| `deliverTelegram` — chat not found → `invalidIdentifier` | Task 3 |
| `/cal notify on|off` Slack subcommand | Task 8 |
| `/notify on|off` Telegram command | Task 9 |
| `TELEGRAM_COMMANDS` array update (auto-updates regex) | Task 9 |
| `registerSlack/Telegram/removeSlack/Telegram` client methods | Task 7 |
| `CALCOM_DELIVERY_SECRET` env validation (hard fail prod, warn dev) | Task 6 |
| `.env.example` documentation | Task 6 |
