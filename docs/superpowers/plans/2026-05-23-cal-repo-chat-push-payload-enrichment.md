# Cal Repo: Enrich ChatPushPayload with Structured Booking Fields

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enrich `ChatPushPayload` in the `/cal` repo with structured booking fields (hosts, attendees, start/end times, timezone, location, meetingUrl, cancellationReason, notificationType) so the companion chat app can render rich push notifications without making any additional API calls at delivery time.

**Architecture:** Option A — `/cal` populates all booking data into the payload at dispatch time. `PrismaBookingPushQueryRepository` fetches the extra fields in one Prisma query. `BookingPushTaskService` passes them to `BookingNotificationDispatchService`, which builds the enriched `ChatPushPayload`. The companion chat app receives everything it needs in the single `POST /api/notifications/deliver` request.

**Tech Stack:** TypeScript, Prisma, Vitest. Repo: `cal` (branch `devin/1779470463-chat-push-notifications`). All paths are relative to the repo root `/Users/dhairyashilshinde/work/calcom/cal`.

---

## Context — Why This PR Exists

The companion chat app (a separate repo) will send Slack DMs and Telegram messages when Cal.com booking events occur. The flow is:

1. A booking lifecycle event triggers `BookingNotificationDispatchService.dispatch()`
2. The service fetches Slack/Telegram subscribers and calls the chat app at `POST /api/notifications/deliver`
3. The chat app renders a Graphite-style compact notification and sends the DM/message

For the chat app to render this notification:
```
📅 *Bi-Weekly Morale Talk zwischen David Borenius und dhairyashil*
Wed Jun 3 · 4:00–4:30 PM IST
David Borenius · david@cal.com, Dhairyashil Shinde · dhairyashil@cal.com
Cal Video: https://app.cal.com/video/kfd...
[View Booking]
```
…it needs `notificationType`, `hosts`, `attendees`, `start`, `end`, `timeZone`, `location`, and `meetingUrl` — none of which are in the current `ChatPushPayload`. This plan adds them.

---

## File Map

| File | Change |
|------|--------|
| `packages/features/notifications/send-chat-push-notification.ts` | Add structured fields to `ChatPushPayload` type |
| `packages/features/notifications/prisma-booking-push-query-repository.ts` | Expand `BookingForPushDispatch` type + Prisma select |
| `packages/features/notifications/dispatch-booking-notification.ts` | Expand `DispatchBookingNotificationInput.booking` type; build enriched `chatPayload`; fix `payload.data ?? {}` |
| `packages/features/notifications/chat-push-subscription-repository.ts` | Fix `satisfies` → `as NotificationPlatform` |
| `packages/features/notifications/tasker/booking-push-task-service.ts` | Pass new booking fields to `dispatchService.dispatch()` |
| `packages/features/notifications/__tests__/dispatch-booking-notification.test.ts` | Update `makeInput()`, fix test quality issues, add enriched-payload assertions |

---

## Task 1 — Fix Remaining Code Quality Issues

These are small correctness bugs identified in code review. Fix them first, before touching any feature code.

**Files:**
- Modify: `packages/features/notifications/chat-push-subscription-repository.ts:25`
- Modify: `packages/features/notifications/dispatch-booking-notification.ts:170-175`
- Modify: `packages/features/notifications/__tests__/dispatch-booking-notification.test.ts`

### Background

`satisfies` is a TypeScript type assertion operator — it checks that a value satisfies a type at compile time but **does not produce a runtime cast**. Using it as a value (`:` in an object literal) is a TypeScript error because `satisfies` is a statement-level expression, not a value. Use `as` instead.

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

Find lines 168–175 (inside `dispatch`, where `chatPayload` is constructed):
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

`toHaveBeenCalled()` passes regardless of arguments — it provides no signal if the wrong platform or wrong subscriptions are passed. The tighter assertion documents the exact expected call contract.

---

- [ ] **Step 4: Add Telegram success assertion to allSettled isolation test**

Find the test `"SLACK rejection does not block TELEGRAM success (Promise.allSettled isolation)"` (around line 486). At the bottom, after the `toHaveBeenCalledTimes(2)` assertion, add:
```ts
    expect(mockSendChatPushNotifications).toHaveBeenCalledWith(
      "TELEGRAM",
      [{ identifier: "TG_123", deviceId: null }],
      expect.objectContaining({ title: "Booking Confirmed" }),
      "https://chat.test.cal.com",
      "test-secret"
    );
```

The test previously only checked that the Slack error was logged — it never verified that Telegram actually fired. This assertion closes that gap.

---

- [ ] **Step 5: Run the tests and verify they pass**

From the repo root:
```bash
yarn vitest run packages/features/notifications/__tests__/dispatch-booking-notification.test.ts
```

Expected: All tests pass with no TypeScript errors. If TypeScript is strict about the `as` cast, it will pass — `ChatPushType` is `Extract<NotificationSubscriptionType, "SLACK" | "TELEGRAM">`, which overlaps with `NotificationPlatform`.

---

- [ ] **Step 6: Commit**

```bash
git add packages/features/notifications/chat-push-subscription-repository.ts \
        packages/features/notifications/dispatch-booking-notification.ts \
        packages/features/notifications/__tests__/dispatch-booking-notification.test.ts
git commit -m "fix(notifications): satisfies→as cast, data spread null-safety, tighter test assertions"
```

---

## Task 2 — Enrich `ChatPushPayload` with Structured Booking Fields

Add the structured fields that the chat app needs to render rich notifications. The new fields go on `ChatPushPayload` in `send-chat-push-notification.ts` — this is the single source of truth for the HTTP contract between `/cal` and the chat app.

**Files:**
- Modify: `packages/features/notifications/send-chat-push-notification.ts`

### Background

`ChatPushPayload` is a plain serializable type — it crosses a network boundary as JSON. That's why it lives in `send-chat-push-notification.ts` (the HTTP delivery module) rather than in a domain-layer file. The chat app maintains a **mirror** of this type for its own rendering logic — the two stay in sync manually, so every field added here must also be added in the companion chat app PR.

`notificationType` is typed as `string` (not an enum) because it crosses a service boundary. The chat app and `/cal` independently define their own rendering maps against this string value. Using `string` avoids importing Prisma enums into what should be a lightweight transport type.

---

- [ ] **Step 1: Add structured fields to `ChatPushPayload`**

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
export type ChatPushPayload = {
  title: string;
  body: string;
  data?: Record<string, string>;
  // Structured booking data for rich chat notification rendering.
  // The chat app uses these fields to format the notification without
  // making additional API calls at delivery time.
  notificationType: string;
  hosts: Array<{ name: string; email: string }>;
  attendees: Array<{ name: string; email: string }>;
  start: string;        // ISO-8601 UTC
  end: string;          // ISO-8601 UTC
  timeZone: string;     // IANA timezone — organizer's, used for display formatting
  location?: string;    // address, phone, or meeting URL if no dedicated video link
  meetingUrl?: string;  // video call URL (e.g. Cal Video, Zoom)
  cancellationReason?: string;
};
```

---

- [ ] **Step 2: Verify TypeScript catches the missing fields downstream**

From the repo root:
```bash
yarn tsc --noEmit 2>&1 | grep "send-chat-push\|dispatch-booking\|booking-push-task"
```

Expected output (before fixing the callers):
```
packages/features/notifications/dispatch-booking-notification.ts(168,11): error TS2741: Property 'notificationType' is missing ...
```

If you see TypeScript errors about the new required fields in `dispatch-booking-notification.ts` and/or `booking-push-task-service.ts`, that confirms the type change is propagating correctly. These will be fixed in Tasks 3–4.

---

- [ ] **Step 3: Commit the type definition (before fixing callers)**

```bash
git add packages/features/notifications/send-chat-push-notification.ts
git commit -m "feat(notifications): add structured booking fields to ChatPushPayload contract"
```

---

## Task 3 — Expand the Prisma Query and Booking Type

The `PrismaBookingPushQueryRepository` currently fetches a narrow booking projection — only what was needed for recipient resolution. We need to expand it to include `endTime`, `location`, `cancellationReason`, `user.timeZone`, `attendees.name`, `hosts.user.{name,email}`, and `references.meetingUrl`.

**Files:**
- Modify: `packages/features/notifications/prisma-booking-push-query-repository.ts`

### Background

In Cal.com's Prisma schema:
- `Booking.endTime` — `DateTime`, always set
- `Booking.location` — `String?`, raw location string (can be an address, phone, "integrations:daily", etc.)
- `Booking.cancellationReason` — `String?`
- `Booking.user` — `User?` relation; `User.timeZone` is `String @default("Europe/London")`
- `Booking.attendees` — `Attendee[]`; `Attendee.name` is `String` (always set)
- `Booking.eventType.hosts` — `Host[]`; each `Host` has a `user: User` relation with `name: String?` and `email: String`
- `Booking.references` — `BookingReference[]`; `BookingReference.meetingUrl` is `String?`; we take the first reference that has a non-null `meetingUrl` (this is the video meeting link, e.g. Cal Video, Zoom)

The `BookingForPushDispatch` TypeScript type and the Prisma `select` must be updated together — they are in the same file and kept in sync manually (there is no codegen for query types in this repo).

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

In the same file, find the `findBookingForPushDispatch` method — specifically the `select` object inside `prisma.booking.findUnique`. Replace the entire `select` block:

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
          take: 1,
        },
      },
```

The `where: { meetingUrl: { not: null } }` filter means `references[0]` is always a video meeting reference if present. We `take: 1` because only one video call link is needed for display.

---

- [ ] **Step 3: Verify the type compiles**

```bash
yarn tsc --noEmit 2>&1 | grep "prisma-booking-push-query"
```

Expected: no errors for this file. TypeScript errors will still exist in `booking-push-task-service.ts` because it passes `hosts` without the new `user` shape — that's fixed in Task 4.

---

- [ ] **Step 4: Commit**

```bash
git add packages/features/notifications/prisma-booking-push-query-repository.ts
git commit -m "feat(notifications): expand booking push query to include endTime, location, attendee names, host users, references"
```

---

## Task 4 — Wire Enriched Payload Through the Dispatch Chain

Now connect the new Prisma fields all the way through to the `chatPayload` object that gets sent to the chat app.

Three files change in this task:
1. `dispatch-booking-notification.ts` — expand `DispatchBookingNotificationInput.booking` type and build enriched `chatPayload`
2. `booking-push-task-service.ts` — pass new fields when calling `dispatchService.dispatch()`

**Files:**
- Modify: `packages/features/notifications/dispatch-booking-notification.ts`
- Modify: `packages/features/notifications/tasker/booking-push-task-service.ts`

### Background

`DispatchBookingNotificationInput.booking` is typed as an intersection of `BookingForNotificationRecipients` (which defines `userId`, `attendees`, `hosts` for recipient resolution) and extra scalar fields (`uid`, `title`, `startTime`). We extend this intersection with the new structured fields needed for chat payload construction.

TypeScript intersection with object types works additively: `{ email: string }[] & { email: string; name: string }[]` resolves to `{ email: string; name: string }[]` — the more specific type satisfies both. So overriding `attendees` and `hosts` in the intersection is safe and does not break `resolveNotificationRecipients`, which only accesses `email` and `userId`.

---

- [ ] **Step 1: Expand `DispatchBookingNotificationInput` booking type**

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
    hosts: Array<{ userId: number; user?: { name?: string | null; email?: string } }>;
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

Note: `attendees` and `hosts` override the same-named fields from `BookingForNotificationRecipients` with more specific shapes (adding `name` and `user`). This is valid TypeScript — the intersection ensures the value satisfies both.

---

- [ ] **Step 2: Build the enriched `chatPayload` in `dispatch`**

In the same file, find the `chatPayload` construction (lines ~168–175):
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
          const chatPayload: ChatPushPayload = {
            title: payload.title,
            body: payload.body,
            data: {
              ...(payload.data ?? {}),
              url: `https://app.cal.com/bookings/${booking.uid}`,
            },
            notificationType: event,
            hosts: booking.hosts.flatMap((h) => {
              const email = h.user?.email;
              if (!email) return [];
              return [{ name: h.user?.name ?? "", email }];
            }),
            attendees: booking.attendees.map((a) => ({ name: a.name, email: a.email })),
            start: booking.startTime.toISOString(),
            end: booking.endTime.toISOString(),
            timeZone: booking.timeZone,
            ...(booking.location ? { location: booking.location } : {}),
            ...(booking.meetingUrl ? { meetingUrl: booking.meetingUrl } : {}),
            ...(booking.cancellationReason ? { cancellationReason: booking.cancellationReason } : {}),
          };
```

Using `flatMap` as a combined filter+map lets TypeScript narrow the type correctly: after the `if (!email) return []` guard, the `email` variable is narrowed to `string` in the return branch. No non-null assertions needed. Hosts without a resolved `user.email` are skipped — this shouldn't happen in practice but protects against inconsistent data.

---

- [ ] **Step 3: Pass new fields in `BookingPushTaskService.execute`**

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
        location: booking.location ?? undefined,
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

`booking.user?.timeZone ?? "UTC"` — the organizer user can be null if `userId` is null (anonymous booker scenario). UTC is a safe fallback; the chat app will format times in UTC if the timezone is missing context.

`booking.references[0]?.meetingUrl ?? undefined` — `references` is filtered to non-null `meetingUrl` in the Prisma query and limited to `take: 1`, so `references[0]` is either the video meeting reference or absent.

`location ?? undefined` — converts `null` (Prisma nullable) to `undefined` (optional field in TypeScript).

---

- [ ] **Step 4: Check that `hosts` variable includes user data**

In `booking-push-task-service.ts`, `hosts` is assigned from `booking.eventType?.hosts ?? []`. After the Prisma query change in Task 3, `booking.eventType.hosts` is now `Array<{ userId: number; user: { name: string | null; email: string } }>`. The `hosts` variable and the dispatch call automatically get the richer type — no additional change needed.

Verify this compiles:
```bash
yarn tsc --noEmit 2>&1 | grep "booking-push-task\|dispatch-booking"
```

Expected: no errors.

---

- [ ] **Step 5: Commit**

```bash
git add packages/features/notifications/dispatch-booking-notification.ts \
        packages/features/notifications/tasker/booking-push-task-service.ts
git commit -m "feat(notifications): wire enriched structured booking data into ChatPushPayload"
```

---

## Task 5 — Update Tests to Verify Enriched Payload

Now update `dispatch-booking-notification.test.ts` to verify that the enriched fields are actually passed through to `sendChatPushNotifications`. This is the regression net — if someone later removes a field from the `chatPayload` construction, these tests will catch it.

**Files:**
- Modify: `packages/features/notifications/__tests__/dispatch-booking-notification.test.ts`

### Background

The test uses a `makeInput()` factory that builds a minimal `DispatchBookingNotificationInput`. After Task 4, `DispatchBookingNotificationInput.booking` now requires `endTime` and `timeZone` — the current `makeInput()` will fail TypeScript. We update it to supply all required fields, then add assertions.

The tests mock `buildBookingNotificationPayload` to return `{ title: "Booking Confirmed", body: "Test event", data: { url: "calcom://test" } }`. That mock stays as-is — we're testing the `chatPayload` construction in `dispatch`, not the payload builder.

---

- [ ] **Step 1: Update `makeInput()` with new required fields**

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

- [ ] **Step 2: Run existing tests to confirm they still pass**

```bash
yarn vitest run packages/features/notifications/__tests__/dispatch-booking-notification.test.ts
```

Expected: all existing tests pass. The new fields have safe defaults (empty arrays for hosts/attendees, so no data to enrich).

---

- [ ] **Step 3: Write a failing test — enriched payload fields are passed to `sendChatPushNotifications`**

Add this test inside the `"BookingNotificationDispatchService - Chat Push"` describe block, right before the closing `});`:

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
          cancellationReason: undefined,
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
    expect(calledPayload.cancellationReason).toBeUndefined();
  });
```

You'll also need to import the `ChatPushPayload` type at the top of the file. Find the imports section (around line 81–88) and add:
```ts
import type { ChatPushPayload } from "../send-chat-push-notification";
```

---

- [ ] **Step 4: Run to confirm the new test fails (before implementation is complete)**

```bash
yarn vitest run packages/features/notifications/__tests__/dispatch-booking-notification.test.ts 2>&1 | tail -20
```

Expected: `FAIL` with something like `expect(received).toBe("BOOKING_CONFIRMED")` or a property access error, confirming the new test is actually exercising the code path.

> **Note:** If you're doing Tasks 2–5 sequentially in this PR, the implementation is already in place from Tasks 2–4, so the test may pass immediately. If it does, skip to Step 5 — that's fine.

---

- [ ] **Step 5: Write a test for cancellationReason — included when present, omitted when absent**

Add this test in the same describe block:

```ts
  it("includes cancellationReason in chatPayload when present, omits it when absent", async () => {
    vi.mocked(resolveNotificationRecipients).mockReturnValue([{ userId: 1, reason: "organizer" }]);
    mockSlackFindByUserIds.mockResolvedValue([{ id: 1, userId: 1, identifier: "U123", deviceId: "T456" }]);
    mockSendChatPushNotifications.mockResolvedValue([{ identifier: "U123", success: true }]);

    const service = createServiceWithChat();

    // With cancellation reason
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
          hosts: [],
          cancellationReason: "Client rescheduled",
        },
      })
    );

    const payloadWithReason = mockSendChatPushNotifications.mock.calls[0]?.[2] as ChatPushPayload;
    expect(payloadWithReason.cancellationReason).toBe("Client rescheduled");

    vi.clearAllMocks();
    mockSendChatPushNotifications.mockResolvedValue([{ identifier: "U123", success: true }]);
    mockSlackFindByUserIds.mockResolvedValue([{ id: 1, userId: 1, identifier: "U123", deviceId: "T456" }]);

    // Without cancellation reason
    await service.dispatch(makeInput());

    const payloadWithoutReason = mockSendChatPushNotifications.mock.calls[0]?.[2] as ChatPushPayload;
    expect(payloadWithoutReason.cancellationReason).toBeUndefined();
  });
```

---

- [ ] **Step 6: Run all tests to confirm they pass**

```bash
yarn vitest run packages/features/notifications/__tests__/dispatch-booking-notification.test.ts
```

Expected output ends with:
```
Test Files  1 passed (1)
Tests       XX passed (XX)
```

If any test fails, read the error carefully — it will point at the specific assertion that's wrong. Common issues:
- `calledPayload.notificationType` is undefined → the `chatPayload` construction in Task 4 Step 2 did not run (check that `chatPushEnabled` flag is true in the test's `beforeEach`)
- `calledPayload.hosts` is `[]` → check the `filter` in the `chatPayload` builder; `h.user?.email` must be truthy

---

- [ ] **Step 7: Commit**

```bash
git add packages/features/notifications/__tests__/dispatch-booking-notification.test.ts
git commit -m "test(notifications): verify enriched ChatPushPayload fields are populated at dispatch time"
```

---

## Final Verification

- [ ] **Run the full test suite one more time**

```bash
yarn vitest run packages/features/notifications/__tests__/dispatch-booking-notification.test.ts
```

Expected: all tests pass.

- [ ] **TypeScript clean build**

```bash
yarn tsc --noEmit 2>&1 | grep -E "notifications/(send-chat|dispatch-booking|chat-push-sub|prisma-booking|booking-push-task)"
```

Expected: no output (no errors in the changed files).

---

## What the Companion Chat App PR Must Also Do

This plan covers only the `/cal` repo side. The companion chat app PR (`apps/chat/`) must implement the **matching changes** in a separate PR:

1. **Mirror `ChatPushPayload` type** — define the same structured fields in `apps/chat/lib/push-notifications.ts` (new file) or `apps/chat/lib/notifications.ts`
2. **`POST /api/notifications/deliver` route** — receive the enriched payload and send the notification
3. **`/cal notifications-on` / `/cal notifications-off` slash commands** — register/remove Slack subscriptions via the `/cal` API
4. **`/notifications-on` / `/notifications-off` Telegram commands** — same for Telegram
5. **Compact notification formatter** — render the Graphite-style message using the structured fields

The companion app PR is tracked separately and should reference this enriched payload contract when implementing the delivery endpoint.
