# Paul — GPS Action Network! link feed integration

This doc gives you everything needed to read the live feed of links shared in
**GPS Action Network!** WhatsApp group from the GPS app, without having to
touch the WhatsApp side.

The pipe is already running (Whapi → Supabase → curated public view). Your
job is the last leg: read from Supabase and render.

---

## TL;DR

```ts
// Server-side, e.g. inside a tRPC procedure
import { createClient } from '@supabase/supabase-js';

const sb = createClient(
  'https://wsvxocihxrhuxtlysgua.supabase.co',
  process.env.GPS_SUPABASE_ANON_KEY!  // value below
);

export const recentLinks = publicProcedure.query(async () => {
  const { data, error } = await sb
    .from('gps_group_messages')
    .select('id, sent_at, from_name, url, link_title, text_body, sender_hash')
    .order('sent_at', { ascending: false })
    .limit(50);
  if (error) throw error;
  return data ?? [];
});
```

`GPS_SUPABASE_ANON_KEY`:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6IndzdnhvY2loeHJodXh0bHlzZ3VhIiwicm9sZSI6ImFub24iLCJpYXQiOjE3Nzc3MDE2OTQsImV4cCI6MjA5MzI3NzY5NH0.177v6sA16JOy-_REetxea82bKnnjn5Pg6xW-16fIfzI
```

---

## What you're reading

Public Postgres view: **`public.gps_group_messages`**

Hosted on the AIFA `agent-state` Supabase project (project ref:
`wsvxocihxrhuxtlysgua`, region `eu-west-1`). The Whapi.cloud ingest pipeline
lives in the `gps` schema; this view is the public read interface.

### Columns (post-9-May revision)

| column | type | notes |
|---|---|---|
| `id` | bigint | sequential, useful for cursors / dedup |
| `sent_at` | timestamptz | original WhatsApp send time. **Use this for ordering / display.** |
| `received_at` | timestamptz | when our pipe stored it (only differs from `sent_at` for backfilled rows) |
| `chat_id` | text | the source group's `@g.us` id |
| `sender_hash` | text | SHA-256 of the raw JID — stable per-sender, opaque. Use for grouping ("links from same person") without seeing the identifier itself. |
| `from_name` | text | display name (NULL when WhatsApp privacy mode hides the sender — common; see "Quirks") |
| `link_title` | text | page title from WhatsApp's link preview, when available |
| `url` | text | extracted URL — guaranteed non-null in this view |
| `text_body` | text | original message text (often just the URL) |
| `message_type` | text | `text`, `link_preview`, etc. |
| `timestamp_unix` | bigint | sent_at as unix epoch, for clients that prefer ints |

### What's NOT in this view (intentional)

- **`from_jid`** — the raw JID (phone number for some senders) is PII-adjacent; not exposed to anon. Use `sender_hash` for grouping. If you ever need contact-resolution beyond what `from_name` provides, that's a server-side join Grant can run.
- **`raw`** — the full Whapi payload. Could contain sender JIDs, profile metadata, and other fields we don't want anon-readable. Available to Grant on the server side if a future feature needs it.

There's also a small lookup view **`public.gps_chat_labels`** (chat_id →
friendly label) for showing source-group names.

### Filter rules already applied (server-side, before this view)

By the time a row is in this view it's already been:
- Allowlisted to in-scope chats (currently just GPS Action Network! + a test group)
- Filtered to messages containing a URL (no plain chatter)
- **Curated** — Grant runs a `curate.py` CLI that hides irrelevant links (e.g. all 14 Zoom meeting invites are hidden as of 9 May). The view filters `WHERE hidden = false` so curation reflects to your client immediately.

You don't need to filter on your end. Every row should be a renderable card.

### Volume

163 rows total today (149 visible after curation), backfilled from
GPS Action Network! history going back a few months. Ongoing rate is
~5-10 new link shares per day (your mileage may vary as the group grows).

---

## Recommended access pattern

**Polling, server-side, every 30-60 seconds.** This is the right shape for
your architecture and our volume.

### Why not Realtime?

- Supabase Realtime emits row changes from **tables**, not views. To use it
  you'd subscribe to the underlying `gps.messages` table, which would bypass
  every filter the view applies (allowlist, link presence, curation, hidden
  state) plus deliver the un-extracted column shape.
- At ~10 inserts/day, polling every 30-60s is invisible-to-user latency and
  saves you a WebSocket dependency.
- If Jeremy ever asks for "live" updates, the right answer is a trigger-fed
  mirror table (a `gps.messages_public` whose schema matches the view shape,
  populated on insert/update) — that's a job for the publisher side, not yours.
  Flag it and Grant builds it.

### Why server-side proxy

You already do this — the anon key never touches the client bundle, your tRPC
layer enforces rate limits / your own auth / response shaping. That's
architecturally cleaner than the client-direct pattern in the original draft.
The anon key is *safe* to embed in clients (audited; can only read this view
+ chat-label view + execute the Whapi ingest function), but server-side is
strictly better.

### Sample tRPC-shaped procedure

```ts
// app/server/routers/gps.ts
import { z } from 'zod';
import { createClient } from '@supabase/supabase-js';
import { router, publicProcedure } from '../trpc';

const sb = createClient(
  process.env.GPS_SUPABASE_URL!,        // https://wsvxocihxrhuxtlysgua.supabase.co
  process.env.GPS_SUPABASE_ANON_KEY!
);

export const gpsRouter = router({
  recentLinks: publicProcedure
    .input(z.object({
      limit: z.number().int().min(1).max(200).default(50),
      cursor: z.number().int().optional(),  // last id seen, for pagination
    }))
    .query(async ({ input }) => {
      let q = sb.from('gps_group_messages')
        .select('id, sent_at, from_name, sender_hash, url, link_title, text_body, message_type')
        .order('sent_at', { ascending: false })
        .limit(input.limit);
      if (input.cursor) q = q.lt('id', input.cursor);
      const { data, error } = await q;
      if (error) throw new TRPCError({ code: 'INTERNAL_SERVER_ERROR', cause: error });
      return data;
    }),
});
```

Put a 30-60s revalidation on the client side (or use React Query's polling
with `refetchInterval`).

---

## Suggested card UI

Each row is one link shared in the group. Render as a Kanban / Feed card with:

- **Title bar:** `link_title` (if present), else the URL hostname
- **Click target:** the `url` (open in new tab)
- **Meta row:** `from_name` (or "anonymous member" if null) · group label · relative time from `sent_at`
- **Body:** `text_body` if it's not just the URL itself

There's a working reference at <https://someshds.github.io/gps-link-feed/>
— single static file, ~270 lines, rendering exactly this. Lift wholesale or
use as a styling reference.

---

## Quirks to know

1. **`from_name` is often null.** WhatsApp introduced `@lid` (lightweight
   identifier) in 2024 — group members appear behind opaque IDs unless saved
   in the burner phone's contacts. About 29% of senders have names; 70% are
   `@lid`. Plan for "anonymous member" being a common case in the UI.
   `sender_hash` is stable across messages from the same `@lid` so you can
   visually group "this anon member shared 3 links this week" without
   knowing who they are.

2. **`sent_at` vs `received_at`.** `sent_at` is the actual WhatsApp send time
   and is what you want to display. `received_at` is when our pipe wrote the
   row — for backfilled history (everything before 9 May 2026), `received_at`
   is all today, which would lie. Always sort/display by `sent_at`.

3. **Dedup is on Whapi message ID** (server-side). The same message won't
   appear twice even if the webhook fires multiple times.

4. **Curation is one-way + reversible** — Grant can hide links at any time;
   they'll disappear from your queries within one polling cycle. He can also
   unhide. So your UI should be eventually-consistent, not hold permanent
   state on `id` membership.

5. **The view is read-only by design.** If you want to mark cards as
   "triaged" or move them on a Kanban, store that workflow state in your
   own table (e.g. `gps_card_state(message_id, status, owner)`) and join.
   Own the workflow state, don't write back.

---

## Adding more groups (for later)

When Jeremy wants more than just GPS Action Network! in scope, Grant runs
`scripts/manage_allowlist.py add <chat_id> "<label>"` on his side and the new
group's messages flow through automatically. No change in your app.

---

## Owner / contact

Grant De Swardt — `grant@aifusionautomations.com`. The pipe + this Supabase
view are owned by AIFA. If something breaks at the WhatsApp / Whapi / DB
layer, that's Grant's side. Above the view is yours.
