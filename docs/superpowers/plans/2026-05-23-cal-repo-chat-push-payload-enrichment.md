# Cal Repo: Enrich ChatPushPayload with Structured Booking Fields

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enrich `ChatPushPayload` with structured booking fields so the companion chat app can render rich Slack/Telegram push notifications (host names, attendee names, formatted time, meeting link) without making any additional API calls at delivery time.

**Architecture:** Option A — `/cal` populates all booking data at dispatch time. `PrismaBookingPushQueryRepository` fetches the extra fields in one Prisma query. `BookingPushTaskService` normalizes them and passes them to `BookingNotificationDispatchService`, which delegates chat payload construction to a new pure function `buildChatPushPayload()` — keeping `dispatch()` free of domain mapping logic. The companion chat app receives everything it needs in the single `POST /api/notifications/deliver` request.

**Tech Stack:** TypeScript, Prisma, Vitest. Repo: `cal` (branch `devin/1779470463-chat-push-notifications`). All paths are relative to the repo root `/Users/dhairyashilshinde/work/calcom/cal`.

---

## Context — Why This PR Exists

The companion chat app (a separate repo) sends Slack DMs and Telegram messages when Cal.com booking events occur. The flow is:

1. A booking lifecycle event triggers `BookingNotificationDispatchService.dispatch()`
2. The service fetches Slack/Telegram subscribers and calls the chat app at `POST /api/notifications/deliver`
3. The chat app renders a Graphite-style compact notification and sends the DM/message

For the chat app to render:
```
📅 *Bi-Weekly Morale Talk zwischen David Borenius und dhairyashil*
Wed Jun 3 · 4:00–4:30 PM IST
David Borenius · david@cal.com, Dhairyashil Shinde · dhairyashil@cal.com
Cal Video: https://app.cal.com/video/kfd...
[View Booking]
```
…it needs structured booking fields — none of which are in the current `ChatPushPayload`.

**Important:** `Booking.location` in Prisma stores raw strings like `"integrations:daily"`, `"integrations:google:meet"`, `"integrations:office365_video"` for video app bookings. These must be normalized to `undefined` before inclusion in the payload — the `meetingUrl` from `BookingReference` is the correct field for video links.

**Note on rescheduling reason:** The Prisma `Booking` model has no `rescheduleReason` field. Only `fromReschedule` (a UID pointing to the previous booking) exists. Surfacing a rescheduling reason string is not possible from the current schema and is out of scope for this PR.

---

## File Map

| File | Change |
|------|--------|
| `packages/features/notifications/send-chat-push-notification.ts` | Add `ChatNotificationType` union + structured fields to `ChatPushPayload` |
| `packages/features/notifications/build-booking-notification-payload.ts` | Add `BookingChatContext` type + `buildChatPushPayload()` pure function |
| `packages/features/notifications/prisma-booking-push-query-repository.ts` | Expand `BookingForPushDispatch` type + Prisma select (with `orderBy`, `take` bounds) |
| `packages/features/notifications/dispatch-booking-notification.ts` | Expand `DispatchBookingNotificationInput.booking` type; call `buildChatPushPayload()` one-liner; fix `payload.data ?? {}`; fix `satisfies`→`as` |
| `packages/features/notifications/chat-push-subscription-repository.ts` | Fix `satisfies` → `as NotificationPlatform` |
| `packages/features/notifications/tasker/booking-push-task-service.ts` | Pass new booking fields + normalize location before dispatch |
| `packages/features/notifications/__tests__/dispatch-booking-notification.test.ts` | Update `makeInput()`, fix test quality issues, add enriched-payload assertions |

---

## Task 1 — Fix Remaining Code Quality Issues

Small correctness bugs from prior code review. Fix these first, in isolation, before touching any feature code.

**Files:**
- Modify: `packages/features/notifications/chat-push-subscription-repository.ts:25`
- Modify: `packages/features/notifications/dispatch-booking-notification.ts:170-175`
- Modify: `packages/features/notifications/__tests__/dispatch-booking-notification.test.ts`

### Background

`satisfies` is a TypeScript type assertion operator — it checks that a value satisfies a type at compile time but **does not produce a runtime cast**. Using it on the right-hand side of a property (`:`) in an object literal is a TypeScript error. Use `as` instead.

`...payload.data` without `?? {}` crashes at runtime when `payload.data` is `undefined` because spread of `undefined` throws in strict mode.

---

- [ ] **Step 1: Fix `satisfies` → `as` in repository**

Open `packages/features/notifications/chat-push-subscription-repository.ts`.

Find line 25 (inside the `create:` object in `upsert`):
```ts
        platform: this.type satisfies NotificationPlatform,
```

Replace with:
```ts
        platform: this.type as NotificationPlatform,
```

The `as` cast is correct here: `this.type` is `ChatPushType` (a subtype of `NotificationSubscriptionType`), and we need to widen it to `NotificationPlatform` for the Prisma insert. TypeScript won't infer this automatically because they are different enums with overlapping string values.

---

- [ ] **Step 2: Fix `payload.data` spread (null-safety)**

Open `packages/features/notifications/dispatch-booking-notification.ts`.

Find lines 168–175 (inside `dispatch`, where `chatPayload` is currently constructed):
```ts
          const chatPayload: ChatPushPayload = {
            title: payload.title,
            body: payload.body,
            data: {
              ...payload.data,
              url: `https://app.cal.com/bookings/${booking.uid}`,
            },
          };
```

Replace with:
```ts
          const chatPayload: ChatPushPayload = {
            title: payload.title,
            body: payload.body,
            data: {
              ...(payload.data ?? {}),
              url: `https://app.cal.com/bookings/${booking.uid}`,
            },
          };
```

> **Note:** This `chatPayload` construction will be fully replaced in Task 5 once `buildChatPushPayload()` exists. The `?? {}` fix is a correct intermediate state — it makes the current code safe until Task 5 lands.

---

- [ ] **Step 3: Tighten independence test assertion**

Open `packages/features/notifications/__tests__/dispatch-booking-notification.test.ts`.

Find the test `"chat dispatch is independent of APP_PUSH eligibility"` (around line 384). At the bottom of that test, find:
```ts
    expect(mockSendChatPushNotifications).toHaveBeenCalled();
```

Replace with:
```ts
    expect(mockSendChatPushNotifications).toHaveBeenCalledWith(
      "SLACK",
      [{ identifier: "U123", deviceId: "T456" }],
      expect.objectContaining({ title: "Booking Confirmed" }),
      "https://chat.test.cal.com",
      "test-secret"
    );
```

---

- [ ] **Step 4: Add Telegram success assertion to allSettled isolation test**

Find the test `"SLACK rejection does not block TELEGRAM success (Promise.allSettled isolation)"` (around line 486). After the `toHaveBeenCalledTimes(2)` assertion, add:
```ts
    expect(mockSendChatPushNotifications).toHaveBeenCalledWith(
      "TELEGRAM",
      [{ identifier: "TG_123", deviceId: null }],
      expect.objectContaining({ title: "Booking Confirmed" }),
      "https://chat.test.cal.com",
      "test-secret"
    );
```

---

- [ ] **Step 5: Run the tests**

```bash
yarn vitest run packages/features/notifications/__tests__/dispatch-booking-notification.test.ts
```

Expected: All tests pass.

---

- [ ] **Step 6: Commit**

```bash
git add packages/features/notifications/chat-push-subscription-repository.ts \
        packages/features/notifications/dispatch-booking-notification.ts \
        packages/features/notifications/__tests__/dispatch-booking-notification.test.ts
git commit -m "fix(notifications): satisfies→as cast, data spread null-safety, tighter test assertions"
```

---

## Task 2 — Enrich `ChatPushPayload` and Add `ChatNotificationType`

Add structured fields to `ChatPushPayload` and define `ChatNotificationType` as a string literal union (not bare `string`). The union gives the companion app compile-time safety when mapping notification types to emojis/rendering logic.

**Files:**
- Modify: `packages/features/notifications/send-chat-push-notification.ts`

### Background

`ChatPushPayload` is a transport type — it crosses a network boundary as JSON. That's why it lives in the delivery module rather than a domain file. The companion app defines a **mirror** of `ChatNotificationType` independently, keyed to the same string values.

Using a string literal union instead of `string`:
- The companion app's exhaustive switch on `notificationType` will get a TypeScript error if a new event is added but not handled
- Callers get autocomplete instead of an open string field

`notificationType` is typed as `ChatNotificationType` here and cast from `NotificationEvent` in the builder (Task 3). Both are string enums with matching values — no runtime risk.

**Note:** `reschedulingReason` is intentionally absent from the type. The Prisma `Booking` model has no rescheduling reason field — only `fromReschedule` (a UID). This is out of scope for this PR.

---

- [ ] **Step 1: Update `send-chat-push-notification.ts`**

Open `packages/features/notifications/send-chat-push-notification.ts`.

Find the current `ChatPushPayload` type (lines 3–7):
```ts
export type ChatPushPayload = {
  title: string;
  body: string;
  data?: Record<string, string>;
};
```

Replace with:
```ts
// All NotificationEvent values that the chat app can receive.
// The companion app mirrors this union independently for its rendering map.
export type ChatNotificationType =
  | "BOOKING_CONFIRMED"
  | "BOOKING_REQUESTED"
  | "BOOKING_RESCHEDULED"
  | "BOOKING_CANCELLED"
  | "BOOKING_REJECTED";

export type ChatPushPayload = {
  title: string;
  body: string;
  data?: Record<string, string>;
  // Structured booking data for rich notification rendering.
  // Populated at dispatch time so the chat app needs no additional API calls.
  notificationType: ChatNotificationType;
  hosts: Array<{ name: string; email: string }>;
  attendees: Array<{ name: string; email: string }>;
  start: string;       // ISO-8601 UTC
  end: string;         // ISO-8601 UTC
  timeZone: string;    // IANA — organizer's timezone, used for display formatting
  location?: string;   // physical address or phone number (video links → use meetingUrl)
  meetingUrl?: string; // video call URL (Cal Video, Zoom, Google Meet, etc.)
  cancellationReason?: string;
};
```

---

- [ ] **Step 2: Verify TypeScript errors propagate downstream**

```bash
yarn tsc --noEmit 2>&1 | grep "dispatch-booking\|build-booking"
```

Expected: TypeScript errors about `notificationType`, `hosts`, etc. missing in `dispatch-booking-notification.ts`. This confirms the type change is forcing the callers to be updated in Tasks 3–5.

---

- [ ] **Step 3: Commit**

```bash
git add packages/features/notifications/send-chat-push-notification.ts
git commit -m "feat(notifications): add ChatNotificationType union and structured fields to ChatPushPayload"
```

---

## Task 3 — Extract `buildChatPushPayload()` Pure Function

Create a pure builder function for the chat payload, co-located with `buildBookingNotificationPayload` in `build-booking-notification-payload.ts`. This keeps all domain-to-payload mapping in one place and removes field-mapping logic from `dispatch()`.

**Files:**
- Modify: `packages/features/notifications/build-booking-notification-payload.ts`

### Background

`dispatch()` already delegates base payload construction to `buildBookingNotificationPayload()`. Putting the chat payload construction inline in `dispatch()` breaks that pattern and mixes domain mapping with orchestration. Extracting `buildChatPushPayload()` means:

- `dispatch()` calls two one-liners: `buildBookingNotificationPayload()` → `buildChatPushPayload()`
- All booking field → payload mapping lives in one testable file
- Future field additions (e.g. meeting password) touch only the Prisma query and this builder — not `dispatch()`

`BookingChatContext` is a new type for the booking data the builder needs. It intentionally does NOT include `userId` or `eventType` — only display fields. This makes the boundary explicit.

---

- [ ] **Step 1: Add imports to `build-booking-notification-payload.ts`**

Open `packages/features/notifications/build-booking-notification-payload.ts`.

The file currently imports:
```ts
import type { NotificationEvent } from "@calcom/prisma/enums";
import type { AppPushPayload } from "./send-app-push-notification";
```

Add the `ChatPushPayload` and `ChatNotificationType` imports:
```ts
import type { NotificationEvent } from "@calcom/prisma/enums";
import type { AppPushPayload } from "./send-app-push-notification";
import type { ChatNotificationType, ChatPushPayload } from "./send-chat-push-notification";
```

---

- [ ] **Step 2: Add `BookingChatContext` type and `buildChatPushPayload()` function**

At the end of the file (after `buildBookingNotificationPayload`), add:

```ts
/** Booking display fields needed to build a rich chat push notification. */
export type BookingChatContext = {
  uid: string;
  startTime: Date;
  endTime: Date;
  timeZone: string;
  location?: string;
  meetingUrl?: string;
  cancellationReason?: string;
  hosts: Array<{ user: { name: string | null; email: string } }>;
  attendees: Array<{ email: string; name: string }>;
};

/**
 * Build a ChatPushPayload for a booking lifecycle event.
 * Merges the base APP_PUSH payload (title, body) with structured booking
 * display fields so the chat app can render without additional API calls.
 */
export function buildChatPushPayload(
  event: NotificationEvent,
  basePayload: AppPushPayload,
  booking: BookingChatContext
): ChatPushPayload {
  return {
    title: basePayload.title,
    body: basePayload.body,
    data: {
      ...(basePayload.data ?? {}),
      url: `https://app.cal.com/bookings/${booking.uid}`,
    },
    notificationType: event as ChatNotificationType,
    hosts: booking.hosts.flatMap((h) => {
      const { email } = h.user;
      if (!email) return [];
      return [{ name: h.user.name ?? "", email }];
    }),
    attendees: booking.attendees.map((a) => ({ name: a.name, email: a.email })),
    start: booking.startTime.toISOString(),
    end: booking.endTime.toISOString(),
    timeZone: booking.timeZone,
    ...(booking.location ? { location: booking.location } : {}),
    ...(booking.meetingUrl ? { meetingUrl: booking.meetingUrl } : {}),
    ...(booking.cancellationReason ? { cancellationReason: booking.cancellationReason } : {}),
  };
}
```

The `event as ChatNotificationType` cast is safe: `NotificationEvent` is a string enum whose values exactly match the `ChatNotificationType` union. If a new `NotificationEvent` value is added to Prisma that isn't in `ChatNotificationType`, TypeScript will warn at the cast site.

The `hosts` `flatMap` uses `if (!email) return []` to skip any host row where the user's email is absent (shouldn't happen given the required Prisma relation, but guards against inconsistent data).

---

- [ ] **Step 3: Run TypeScript check on the new function**

```bash
yarn tsc --noEmit 2>&1 | grep "build-booking-notification-payload"
```

Expected: no errors for this file. Errors in `dispatch-booking-notification.ts` will remain until Task 5.

---

- [ ] **Step 4: Commit**

```bash
git add packages/features/notifications/build-booking-notification-payload.ts
git commit -m "feat(notifications): extract buildChatPushPayload() pure builder function"
```

---

## Task 4 — Expand the Prisma Query and Booking Type

Expand `PrismaBookingPushQueryRepository` to fetch the additional fields needed by `buildChatPushPayload()`.

**Files:**
- Modify: `packages/features/notifications/prisma-booking-push-query-repository.ts`

### Background

In Cal.com's Prisma schema:
- `Booking.endTime` — `DateTime`, always set
- `Booking.location` — `String?`, raw string (can be `"integrations:daily"`, `"https://meet.google.com/..."`, address, phone, null)
- `Booking.cancellationReason` — `String?`
- `Booking.user` — `User?` relation; `User.timeZone` is `String @default("Europe/London")`
- `Booking.attendees` — `Attendee[]`; `Attendee.name` is `String` (always set)
- `Booking.eventType.hosts` — `Host[]`; each `Host` has a required `user: User` relation with `name: String?` and `email: String`
- `Booking.references` — `BookingReference[]`; `BookingReference.meetingUrl` is `String?`

**`references` design decisions:**
- Filter with `where: { meetingUrl: { not: null } }` to skip non-video references (calendar sync, etc.)
- Add `orderBy: { id: "asc" }` — without this, `take: 1` is non-deterministic when a booking has multiple video references (e.g. rescheduled bookings can accumulate references)
- `take: 1` — one video link is sufficient for display

**`attendees` bound:** Add `take: 20` — no limit is a latency risk for group/webinar events with hundreds of attendees. 20 is sufficient for display.

---

- [ ] **Step 1: Update `BookingForPushDispatch` type**

Open `packages/features/notifications/prisma-booking-push-query-repository.ts`.

Find the current type (lines 4–15):
```ts
/** Booking projection needed for push notification dispatch. */
export type BookingForPushDispatch = {
  id: number;
  uid: string;
  title: string;
  startTime: Date;
  userId: number | null;
  attendees: Array<{ email: string }>;
  eventType: {
    hosts: Array<{ userId: number }>;
    teamId: number | null;
  } | null;
};
```

Replace with:
```ts
/** Booking projection needed for push notification dispatch. */
export type BookingForPushDispatch = {
  id: number;
  uid: string;
  title: string;
  startTime: Date;
  endTime: Date;
  userId: number | null;
  location: string | null;
  cancellationReason: string | null;
  attendees: Array<{ email: string; name: string }>;
  user: { timeZone: string } | null;
  eventType: {
    hosts: Array<{ userId: number; user: { name: string | null; email: string } }>;
    teamId: number | null;
  } | null;
  references: Array<{ meetingUrl: string | null }>;
};
```

---

- [ ] **Step 2: Update the Prisma `select` to match**

In the same file, find the `select` inside `prisma.booking.findUnique`. Replace the full `select` block:

Current:
```ts
      select: {
        id: true,
        uid: true,
        title: true,
        startTime: true,
        userId: true,
        attendees: {
          select: { email: true },
        },
        eventType: {
          select: {
            hosts: {
              select: { userId: true },
            },
            teamId: true,
          },
        },
      },
```

Replace with:
```ts
      select: {
        id: true,
        uid: true,
        title: true,
        startTime: true,
        endTime: true,
        userId: true,
        location: true,
        cancellationReason: true,
        attendees: {
          select: { email: true, name: true },
          take: 20,
        },
        user: {
          select: { timeZone: true },
        },
        eventType: {
          select: {
            hosts: {
              select: {
                userId: true,
                user: { select: { name: true, email: true } },
              },
            },
            teamId: true,
          },
        },
        references: {
          select: { meetingUrl: true },
          where: { meetingUrl: { not: null } },
          orderBy: { id: "asc" },
          take: 1,
        },
      },
```

`take: 20` on attendees caps the result for display. `orderBy: { id: "asc" }` + `take: 1` on references makes the video link deterministic.

---

- [ ] **Step 3: Verify the type compiles**

```bash
yarn tsc --noEmit 2>&1 | grep "prisma-booking-push-query"
```

Expected: no errors for this file. TypeScript errors in `booking-push-task-service.ts` will remain until Task 5.

---

- [ ] **Step 4: Commit**

```bash
git add packages/features/notifications/prisma-booking-push-query-repository.ts
git commit -m "feat(notifications): expand booking push query — endTime, location, attendee names, host users, references"
```

---

## Task 5 — Wire Enriched Payload Through the Dispatch Chain

Connect the new Prisma fields to `buildChatPushPayload()` and normalize the `location` field before it enters the payload.

**Files:**
- Modify: `packages/features/notifications/dispatch-booking-notification.ts`
- Modify: `packages/features/notifications/tasker/booking-push-task-service.ts`

### Background

**Location normalization:** `Booking.location` stores integration keys like `"integrations:daily"`, `"integrations:google:meet"`, `"integrations:office365_video"` when a video app is used. These are internal identifiers — not human-readable strings. The chat app must never display them as location text; the `meetingUrl` from references is the correct field. The normalization step (`.startsWith("integrations:")` → `undefined`) is placed in `BookingPushTaskService` before the dispatch call — keeping the dispatch service unaware of this Cal.com-specific quirk.

This exact pattern (`location.type.startsWith("integrations:")`) already exists in `packages/atoms/vendor/locations.ts:189`.

**Type design:** `DispatchBookingNotificationInput.booking` is widened to include the new display fields via TypeScript intersection. `user` in `hosts` is typed as required (not optional) because after Task 4 the Prisma select always returns it. Making it required documents the invariant and removes unnecessary optional chaining in `buildChatPushPayload()`.

---

- [ ] **Step 1: Update `DispatchBookingNotificationInput` booking type**

Open `packages/features/notifications/dispatch-booking-notification.ts`.

Find the current type (lines 31–42):
```ts
export type DispatchBookingNotificationInput = {
  event: NotificationEvent;
  booking: BookingForNotificationRecipients & {
    uid: string;
    title: string;
    startTime: Date;
  };
  /** Map of lowercase attendee email -> Cal.com user ID */
  attendeeUserIds: Map<string, number>;
  /** Organization ID for preference resolution (null if no org) */
  organizationId: number | null;
};
```

Replace with:
```ts
export type DispatchBookingNotificationInput = {
  event: NotificationEvent;
  booking: BookingForNotificationRecipients & {
    uid: string;
    title: string;
    startTime: Date;
    endTime: Date;
    timeZone: string;
    attendees: Array<{ email: string; name: string }>;
    hosts: Array<{ userId: number; user: { name: string | null; email: string } }>;
    location?: string;
    meetingUrl?: string;
    cancellationReason?: string;
  };
  /** Map of lowercase attendee email -> Cal.com user ID */
  attendeeUserIds: Map<string, number>;
  /** Organization ID for preference resolution (null if no org) */
  organizationId: number | null;
};
```

`attendees` and `hosts` override the narrower shapes in `BookingForNotificationRecipients` via intersection — the wider shapes satisfy the narrower ones, so `resolveNotificationRecipients` (which only reads `attendee.email` and `host.userId`) continues to work without changes.

`user` is typed as required on hosts because the Prisma select in Task 4 always loads it.

---

- [ ] **Step 2: Add `buildChatPushPayload` import**

Find the existing imports at the top of `dispatch-booking-notification.ts`. Add `buildChatPushPayload` and `BookingChatContext` to the import from `build-booking-notification-payload`:

Find:
```ts
import {
  type BookingNotificationContext,
  buildBookingNotificationPayload,
} from "./build-booking-notification-payload";
```

Replace with:
```ts
import {
  type BookingChatContext,
  type BookingNotificationContext,
  buildBookingNotificationPayload,
  buildChatPushPayload,
} from "./build-booking-notification-payload";
```

---

- [ ] **Step 3: Replace inline `chatPayload` construction with one-liner**

Find the `chatPayload` construction block (which was fixed in Task 1 Step 2):
```ts
          const chatPayload: ChatPushPayload = {
            title: payload.title,
            body: payload.body,
            data: {
              ...(payload.data ?? {}),
              url: `https://app.cal.com/bookings/${booking.uid}`,
            },
          };
```

Replace with:
```ts
          const chatPayload = buildChatPushPayload(event, payload, booking as BookingChatContext);
```

`booking` satisfies `BookingChatContext` because `DispatchBookingNotificationInput.booking` is a superset of it (extra fields like `userId`, `title` don't cause issues structurally). The `as BookingChatContext` cast is safe here; if the shapes ever diverge, TypeScript will error.

---

- [ ] **Step 4: Verify `dispatch-booking-notification.ts` compiles cleanly**

```bash
yarn tsc --noEmit 2>&1 | grep "dispatch-booking-notification"
```

Expected: no errors.

---

- [ ] **Step 5: Pass new fields in `BookingPushTaskService.execute` with location normalization**

Open `packages/features/notifications/tasker/booking-push-task-service.ts`.

Find the `dispatchService.dispatch()` call (lines 45–57):
```ts
    const result = await dispatchService.dispatch({
      event: notificationEvent as NotificationEvent,
      booking: {
        uid: booking.uid,
        title: booking.title,
        startTime: booking.startTime,
        userId: booking.userId,
        attendees: booking.attendees,
        hosts,
      },
      attendeeUserIds,
      organizationId,
    });
```

Replace with:
```ts
    const result = await dispatchService.dispatch({
      event: notificationEvent as NotificationEvent,
      booking: {
        uid: booking.uid,
        title: booking.title,
        startTime: booking.startTime,
        endTime: booking.endTime,
        timeZone: booking.user?.timeZone ?? "UTC",
        // Filter out Cal.com integration keys (e.g. "integrations:daily") — meetingUrl covers those.
        location: booking.location?.startsWith("integrations:") ? undefined : (booking.location ?? undefined),
        meetingUrl: booking.references[0]?.meetingUrl ?? undefined,
        cancellationReason: booking.cancellationReason ?? undefined,
        userId: booking.userId,
        attendees: booking.attendees,
        hosts,
      },
      attendeeUserIds,
      organizationId,
    });
```

`booking.user?.timeZone ?? "UTC"` — `user` is null for anonymous bookings (no `userId`). UTC is a safe display fallback; the chat app will still show correct absolute times.

`booking.location?.startsWith("integrations:")` — matches the existing pattern in `packages/atoms/vendor/locations.ts:189`. When true, the video meeting URL is in `booking.references[0]?.meetingUrl` instead.

`booking.references[0]?.meetingUrl ?? undefined` — `references` is filtered to non-null `meetingUrl` and limited to `take: 1` in Task 4, so `references[0]` is the video reference if present.

Note: `?? undefined` after `booking.references[0]?.meetingUrl` is technically redundant (optional chaining already returns `undefined`) but is kept for readability alongside the other null coercions. It can be dropped if the team prefers.

---

- [ ] **Step 6: Verify full compile**

```bash
yarn tsc --noEmit 2>&1 | grep -E "notifications/(dispatch-booking|booking-push-task|prisma-booking|build-booking|send-chat|chat-push-sub)"
```

Expected: no output (no errors in changed files).

---

- [ ] **Step 7: Commit**

```bash
git add packages/features/notifications/dispatch-booking-notification.ts \
        packages/features/notifications/tasker/booking-push-task-service.ts
git commit -m "feat(notifications): wire enriched booking data into ChatPushPayload via buildChatPushPayload()"
```

---

## Task 6 — Update Tests

Update the test suite to cover enriched fields, new edge cases, and the `BookingPushTaskService` wiring.

**Files:**
- Modify: `packages/features/notifications/__tests__/dispatch-booking-notification.test.ts`

### Background

The test suite mocks `buildBookingNotificationPayload` to return `{ title: "Booking Confirmed", body: "Test event", data: { url: "calcom://test" } }`. After Task 3, the dispatch path now calls `buildChatPushPayload()` instead of building inline — but since `buildChatPushPayload` is imported from `build-booking-notification-payload.ts` (not mocked), it runs for real in the test. This is fine — it's a pure function.

**Important:** `makeInput()` spreads `overrides` at the top level — it does NOT deep-merge `booking`. When passing `{ booking: { ... } }` as an override, you must supply ALL required booking fields. The tests below do this explicitly.

---

- [ ] **Step 1: Add `ChatPushPayload` import**

Find the imports section at the top of the test file (around line 81–88). Add:
```ts
import type { ChatPushPayload } from "../send-chat-push-notification";
```

---

- [ ] **Step 2: Update `makeInput()` with new required fields**

Find the `makeInput` function (lines 90–107):
```ts
function makeInput(
  overrides: Partial<DispatchBookingNotificationInput> = {}
): DispatchBookingNotificationInput {
  return {
    event: "BOOKING_CONFIRMED",
    booking: {
      uid: "test-uid-123",
      title: "30 Minute Meeting",
      startTime: new Date("2026-04-10T14:00:00Z"),
      userId: 1,
      attendees: [],
      hosts: [],
    },
    attendeeUserIds: new Map(),
    organizationId: null,
    ...overrides,
  };
}
```

Replace with:
```ts
function makeInput(
  overrides: Partial<DispatchBookingNotificationInput> = {}
): DispatchBookingNotificationInput {
  return {
    event: "BOOKING_CONFIRMED",
    booking: {
      uid: "test-uid-123",
      title: "30 Minute Meeting",
      startTime: new Date("2026-04-10T14:00:00Z"),
      endTime: new Date("2026-04-10T14:30:00Z"),
      timeZone: "America/New_York",
      userId: 1,
      attendees: [],
      hosts: [],
    },
    attendeeUserIds: new Map(),
    organizationId: null,
    ...overrides,
  };
}
```

---

- [ ] **Step 3: Run existing tests to confirm they still pass**

```bash
yarn vitest run packages/features/notifications/__tests__/dispatch-booking-notification.test.ts
```

Expected: all tests pass.

---

- [ ] **Step 4: Write test — enriched fields pass through to `sendChatPushNotifications`**

Add inside the `"BookingNotificationDispatchService - Chat Push"` describe block, before the closing `});`:

```ts
  it("passes enriched structured fields in chatPayload", async () => {
    vi.mocked(resolveNotificationRecipients).mockReturnValue([{ userId: 1, reason: "organizer" }]);
    mockSlackFindByUserIds.mockResolvedValue([{ id: 1, userId: 1, identifier: "U123", deviceId: "T456" }]);
    mockSendChatPushNotifications.mockResolvedValue([{ identifier: "U123", success: true }]);

    const service = createServiceWithChat();
    await service.dispatch(
      makeInput({
        event: "BOOKING_CONFIRMED",
        booking: {
          uid: "test-uid-123",
          title: "30 Minute Meeting",
          startTime: new Date("2026-04-10T14:00:00Z"),
          endTime: new Date("2026-04-10T14:30:00Z"),
          timeZone: "America/New_York",
          userId: 1,
          location: "https://meet.google.com/abc-def",
          meetingUrl: "https://app.cal.com/video/xyz",
          attendees: [{ email: "attendee@example.com", name: "Jane Attendee" }],
          hosts: [{ userId: 99, user: { name: "Host Person", email: "host@example.com" } }],
        },
      })
    );

    const calledPayload = mockSendChatPushNotifications.mock.calls[0]?.[2] as ChatPushPayload;
    expect(calledPayload.notificationType).toBe("BOOKING_CONFIRMED");
    expect(calledPayload.hosts).toEqual([{ name: "Host Person", email: "host@example.com" }]);
    expect(calledPayload.attendees).toEqual([{ name: "Jane Attendee", email: "attendee@example.com" }]);
    expect(calledPayload.start).toBe("2026-04-10T14:00:00.000Z");
    expect(calledPayload.end).toBe("2026-04-10T14:30:00.000Z");
    expect(calledPayload.timeZone).toBe("America/New_York");
    expect(calledPayload.meetingUrl).toBe("https://app.cal.com/video/xyz");
    expect(calledPayload.data?.url).toBe("https://app.cal.com/bookings/test-uid-123");
  });
```

---

- [ ] **Step 5: Write test — `notificationType` reflects the event value**

```ts
  it("notificationType in chatPayload reflects the booking event", async () => {
    vi.mocked(resolveNotificationRecipients).mockReturnValue([{ userId: 1, reason: "organizer" }]);
    mockSlackFindByUserIds.mockResolvedValue([{ id: 1, userId: 1, identifier: "U123", deviceId: "T456" }]);
    mockSendChatPushNotifications.mockResolvedValue([{ identifier: "U123", success: true }]);

    const service = createServiceWithChat();
    await service.dispatch(
      makeInput({
        event: "BOOKING_CANCELLED",
        booking: {
          uid: "test-uid-123",
          title: "30 Minute Meeting",
          startTime: new Date("2026-04-10T14:00:00Z"),
          endTime: new Date("2026-04-10T14:30:00Z"),
          timeZone: "UTC",
          userId: 1,
          cancellationReason: "Client rescheduled",
          attendees: [],
          hosts: [],
        },
      })
    );

    const calledPayload = mockSendChatPushNotifications.mock.calls[0]?.[2] as ChatPushPayload;
    expect(calledPayload.notificationType).toBe("BOOKING_CANCELLED");
    expect(calledPayload.cancellationReason).toBe("Client rescheduled");
  });
```

---

- [ ] **Step 6: Write test — `location` without `meetingUrl`**

```ts
  it("includes location in chatPayload when meetingUrl is absent", async () => {
    vi.mocked(resolveNotificationRecipients).mockReturnValue([{ userId: 1, reason: "organizer" }]);
    mockSlackFindByUserIds.mockResolvedValue([{ id: 1, userId: 1, identifier: "U123", deviceId: "T456" }]);
    mockSendChatPushNotifications.mockResolvedValue([{ identifier: "U123", success: true }]);

    const service = createServiceWithChat();
    await service.dispatch(
      makeInput({
        booking: {
          uid: "test-uid-123",
          title: "30 Minute Meeting",
          startTime: new Date("2026-04-10T14:00:00Z"),
          endTime: new Date("2026-04-10T14:30:00Z"),
          timeZone: "UTC",
          userId: 1,
          location: "Conference Room A, Floor 3",
          attendees: [],
          hosts: [],
        },
      })
    );

    const calledPayload = mockSendChatPushNotifications.mock.calls[0]?.[2] as ChatPushPayload;
    expect(calledPayload.location).toBe("Conference Room A, Floor 3");
    expect(calledPayload.meetingUrl).toBeUndefined();
  });
```

---

- [ ] **Step 7: Write test — host with missing email is silently dropped**

```ts
  it("drops hosts with missing email from chatPayload", async () => {
    vi.mocked(resolveNotificationRecipients).mockReturnValue([{ userId: 1, reason: "organizer" }]);
    mockSlackFindByUserIds.mockResolvedValue([{ id: 1, userId: 1, identifier: "U123", deviceId: "T456" }]);
    mockSendChatPushNotifications.mockResolvedValue([{ identifier: "U123", success: true }]);

    const service = createServiceWithChat();
    await service.dispatch(
      makeInput({
        booking: {
          uid: "test-uid-123",
          title: "30 Minute Meeting",
          startTime: new Date("2026-04-10T14:00:00Z"),
          endTime: new Date("2026-04-10T14:30:00Z"),
          timeZone: "UTC",
          userId: 1,
          attendees: [],
          // Host with empty email — should be filtered out
          hosts: [
            { userId: 1, user: { name: "Valid Host", email: "valid@example.com" } },
            { userId: 2, user: { name: "No Email Host", email: "" } },
          ],
        },
      })
    );

    const calledPayload = mockSendChatPushNotifications.mock.calls[0]?.[2] as ChatPushPayload;
    expect(calledPayload.hosts).toEqual([{ name: "Valid Host", email: "valid@example.com" }]);
  });
```

---

- [ ] **Step 8: Write test — empty hosts and attendees produce empty arrays**

```ts
  it("produces empty hosts and attendees arrays when none are provided", async () => {
    vi.mocked(resolveNotificationRecipients).mockReturnValue([{ userId: 1, reason: "organizer" }]);
    mockSlackFindByUserIds.mockResolvedValue([{ id: 1, userId: 1, identifier: "U123", deviceId: "T456" }]);
    mockSendChatPushNotifications.mockResolvedValue([{ identifier: "U123", success: true }]);

    const service = createServiceWithChat();
    await service.dispatch(makeInput()); // default booking has attendees: [], hosts: []

    const calledPayload = mockSendChatPushNotifications.mock.calls[0]?.[2] as ChatPushPayload;
    expect(calledPayload.hosts).toEqual([]);
    expect(calledPayload.attendees).toEqual([]);
  });
```

---

- [ ] **Step 9: Run all tests**

```bash
yarn vitest run packages/features/notifications/__tests__/dispatch-booking-notification.test.ts
```

Expected output ends with:
```
Test Files  1 passed (1)
Tests       XX passed (XX)
```

If `notificationType` is undefined: confirm `buildChatPushPayload` is imported from `build-booking-notification-payload.ts` in `dispatch-booking-notification.ts` (Task 5 Step 2), and that `vi.mock("../build-booking-notification-payload", ...)` at the top of the test does NOT mock `buildChatPushPayload` (it should only mock `buildBookingNotificationPayload`). If it does mock the whole module, add a `vi.unmock` or adjust the mock factory to pass through `buildChatPushPayload`.

---

- [ ] **Step 10: Commit**

```bash
git add packages/features/notifications/__tests__/dispatch-booking-notification.test.ts
git commit -m "test(notifications): enriched chatPayload fields, edge cases for hosts, location, empty arrays"
```

---

## Final Verification

- [ ] **Full test run**

```bash
yarn vitest run packages/features/notifications/__tests__/dispatch-booking-notification.test.ts
```

Expected: all tests pass.

- [ ] **TypeScript clean build across all changed files**

```bash
yarn tsc --noEmit 2>&1 | grep -E "notifications/(dispatch-booking|booking-push-task|prisma-booking|build-booking|send-chat|chat-push-sub)"
```

Expected: no output.

---

## What the Companion Chat App PR Must Also Do

This plan covers only the `/cal` repo side. The companion chat app PR (`apps/chat/`) must implement the matching changes:

1. **Mirror `ChatNotificationType` union** — define the same string literal values for rendering logic
2. **Mirror `ChatPushPayload` structured fields** — the delivery endpoint receives this shape
3. **`POST /api/notifications/deliver` route** — receive the enriched payload and send the DM/message
4. **Compact notification formatter** — render the Graphite-style message using structured fields
5. **`/cal notifications-on` / `/cal notifications-off`** — Slack subscription commands
6. **`/notifications-on` / `/notifications-off`** — Telegram subscription commands
7. **`CALCOM_CHAT_DELIVERY_SECRET` env var** — validate in `apps/chat/lib/env.ts`
