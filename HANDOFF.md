# BB Car Detailing — Project Handoff & Context

> **Read this first.** This file is the complete state of the BB Car Detailing
> project. A new Claude Code session can read it and continue at full
> capability. Last updated: 2026-06-18.

---

## 1. What this project is

A single-page **mobile car detailing website** for **BB Car Detailing**
(tagline **"Cleaner. Shinier. Better."**). Owners: **Bobby** and **Bramley**.
Dark theme (blacks/greys), mobile-first (customers arrive from
Facebook/Instagram links). The silver "BB Car Detailing" logo lives in the repo
as `logo.jpg`.

> **Naming note:** the business was originally drafted as "BB Details"; it was
> renamed to **BB Car Detailing** to match the logo. You may still see the old
> name in old git history, but the live site and all current text use
> "BB Car Detailing".

The entire website is **one self-contained file, `index.html`** (HTML + CSS +
JavaScript, no build step). Bookings and availability are stored online in
**Supabase**, reached through **one Edge Function**. Hosted free on **GitHub
Pages**.

---

## 2. Where everything lives

| Thing | Location |
| --- | --- |
| **Live website repo** | `github.com/bobbyparnell0109/Booking-system` (public) |
| **Live URL** | https://bobbyparnell0109.github.io/Booking-system/ |
| **Hosting** | GitHub Pages → branch `main`, `/ (root)`. `.nojekyll` is present. |
| **The whole site** | `index.html` at the repo root |
| **Admin/owner page** | live URL + `#admin` — password **`bbdetails2026`** |
| **Backend** | Supabase project (details below) |
| **Dev/source history** | also committed in repo `bobbyparnell0109/my-first-project`, branch `claude/bb-details-website-lbv1j7` (this is an unrelated "Revise" app repo — do NOT touch its other files; BB Details' real home is Booking-system) |

---

## 3. Update workflow (IMPORTANT — read before editing)

The user (Bobby) is **non-technical and usually on an iPhone**. The Booking-system
repo may **not** be in a given session's allowed-repos scope, in which case
Claude **cannot push to it directly**. The established workflow is:

1. Claude edits `index.html` locally and **sends the file to the user**.
2. User uploads it to the Booking-system repo via the **GitHub website in
   Safari (Request Desktop Website)** → **Add file → Upload files** → drag
   `index.html` → **Commit changes**. (The GitHub *mobile app* cannot upload
   files — must use the browser in desktop mode.)
3. After uploading, view the site with a **cache-buster** (`?2`, `?3`, …) on
   the URL, because Safari/GitHub Pages cache aggressively. A plain reload
   often shows the old version — this has confused the user before; reassure
   them it's only their cached copy.

**If a session IS scoped to Booking-system** (as the 2026-06-17 session was),
skip the manual upload entirely — work on the feature branch, commit, and push
directly. Uploaded images (logo/photos) can be copied straight from the
session's upload folder into the repo and committed; the user does **not** need
to upload them via Safari in that case.

**Going live (IMPORTANT):** GitHub Pages serves from the **`main`** branch, so
changes on a feature branch are **not live** until merged into `main`. With the
GitHub tools: open a PR (`head` = feature branch, `base` = main) and merge it
(squash). Only do this when the user says to go live. After merging, give the
user the live URL with a **cache-buster** (`?v=2`, `?v=3`, …). The user has
been happy for Claude to open AND merge the PR itself once they say "go live".

**Squash-merge gotcha (do this every time):** squash-merging rewrites history,
so the feature branch diverges from `main` and the *next* PR can report a
phantom merge conflict. After each successful merge, resync the branch before
the next change:
`git fetch origin main && git reset --hard origin/main && git push -f origin <branch>`.
That keeps the branch a clean continuation of `main`.

**Docs must be on `main`:** a brand-new session clones `main`, so HANDOFF.md /
POSTER-PROMPT.md changes only reach the next session if they're merged to
`main` (not left on the feature branch).

Backend changes (Supabase) are done by Claude directly via the Supabase MCP
tools — no user action needed, and they affect the **live** site immediately
(the function is shared), so coordinate function changes with the matching
front-end merge.

---

## 4. Backend — Supabase

- **Project name:** "bobbyparnell0109's Project"
- **Project ref / id:** `qqkxxywdlqqkipioycaq`
- **API URL:** `https://qqkxxywdlqqkipioycaq.supabase.co`
- **Edge Function:** `bb-bookings`, deployed with **`verify_jwt = false`**
  (it implements its own auth — public actions are open, owner actions check a
  password). Function URL:
  `https://qqkxxywdlqqkipioycaq.supabase.co/functions/v1/bb-bookings`
- **Security model:** both tables have **RLS enabled with NO policies**, so the
  public/anon key cannot read or write them at all. Only the Edge Function
  (using the service role) touches the data, so customer details
  (names/phones/addresses) are never publicly readable. The admin password
  lives **only in the function**, not in the website file.

### Business rules (constants in the Edge Function)
- `JOB_MINUTES = 150` — each valet takes 2½ hours.
- `STEP_MINUTES = 30` — bookable start times every 30 minutes.
- `HORIZON_DAYS = 42` — how many days ahead bookings are offered (6 weeks).
- `PERSONS = ["Bobby", "Bramley"]` — the two detailers.
- `DEFAULT_HOURS` — **weekdays (Mon–Fri) 16:30–19:00** (after school),
  **weekends (Sat/Sun) 10:00–18:00**. Applies to both people automatically.
- `ADMIN_PASSWORD = "bbdetails2026"`.
- To change any of these: edit the function source (Section 7) and redeploy
  via the Supabase MCP `deploy_edge_function` (keep `verify_jwt = false`).

### How availability works (INVERTED model — "book time off", not "add free time")
Both people are assumed **free by default** during `DEFAULT_HOURS` every day.
Owners no longer add free time; instead they book **time off** (holidays / days
they can't work) per person, optionally as a multi-day date range. The function
generates the implicit free ranges for the next `HORIZON_DAYS` days from
`DEFAULT_HOURS`, **skips any day a person has booked off**, then for each
remaining day produces start times every 30 min where `start + 150min <= end`,
and removes any start whose 2½-hour job would **overlap** an existing booking for
that person (overlap test: `t < bookingTime+150 && bookingTime < t+150`).
So a weekday yields a single start (16:30); a weekend yields 10:00–15:30.
If **both** people are free at the same start time, the website combines them
into a single **"Bobby & Bramley (together)"** option; booking it ties up both.

> **History:** the original design used an `availability` table of explicit
> free-time ranges (`add_range`/`list_ranges`/`delete_range`). On 2026-06-17 this
> was inverted to the default-hours + time-off model above. The old
> `availability` table still exists but is **no longer read or written** by the
> function; the new `time_off` table drives blocking.

---

## 5. Database schema

```sql
-- Bookings
create table public.bookings (
  id            uuid primary key default gen_random_uuid(),
  created_at    timestamptz not null default now(),
  name          text not null,
  phone         text not null,
  email         text not null,
  service_id    text,
  service_label text,
  price         numeric,
  booking_date  date not null,
  booking_time  text not null,        -- "HH:MM"
  car           text,
  address       text,
  notes         text,
  completed     boolean not null default false,
  detailer      text,                 -- "Bobby" | "Bramley" | "Bobby & Bramley (together)"
  busy_persons  text[]                -- who the booking ties up, e.g. {"Bobby"} or {"Bobby","Bramley"}
);
alter table public.bookings enable row level security;  -- no policies: function-only

-- Time off (holidays / days a person is unavailable). Drives slot blocking.
create table public.time_off (
  id          uuid primary key default gen_random_uuid(),
  created_at  timestamptz not null default now(),
  person      text not null,          -- "Bobby" | "Bramley"
  start_date  date not null,
  end_date    date not null           -- inclusive; == start_date for a single day
);
alter table public.time_off enable row level security;  -- no policies: function-only

-- Legacy: availability ranges (NO LONGER USED — kept for history, not read/written)
create table public.availability (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz not null default now(),
  person text not null, avail_date date not null,
  start_time text not null, end_time text not null,
  unique (person, avail_date, start_time, end_time)
);
alter table public.availability enable row level security;  -- no policies: function-only
```

---

## 6. Edge Function actions (the website's API)

POST JSON to the function URL with an `action`:

**Public (no password):**
- `{ action: "slots" }` → `{ slots: [{slot_date, slot_time, person}, …] }`
  (open start times; the site groups same date+time across people).
- `{ action: "create", booking: {…}, persons: ["Bobby"] }` → re-validates the
  time is still free for all `persons`, returns `409 {error:"taken"}` if not,
  else inserts and returns `{ok:true}`.

**Owner (must include `password`):**
- `{ action: "list" }` → all bookings
- `{ action: "complete", id, completed }` → toggle completed
- `{ action: "delete", id }` → delete a booking
- `{ action: "list_time_off" }` → upcoming time off rows
- `{ action: "add_time_off", time_off: {person, start_date, end_date?} }` —
  `person` may be `"Both"` (inserts a row for each of `PERSONS`); `end_date`
  optional (defaults to `start_date` for a single day).
- `{ action: "delete_time_off", id }`

---

## 7. Edge Function source (`bb-bookings`, current deployed version)

> If you change this, redeploy with the Supabase MCP `deploy_edge_function`
> (project `qqkxxywdlqqkipioycaq`, name `bb-bookings`, `verify_jwt: false`).

```ts
import "jsr:@supabase/functions-js/edge-runtime.d.ts";

const ADMIN_PASSWORD = "bbdetails2026";  // <-- change me
const JOB_MINUTES = 150;                 // length of one valet (2.5 hours)
const STEP_MINUTES = 30;                 // offer a start time every 30 mins
const HORIZON_DAYS = 42;                 // how many days ahead to offer bookings (6 weeks)
const PERSONS = ["Bobby", "Bramley"];    // the two detailers

// Standard working hours assumed automatically every day. Owners book TIME OFF
// (holidays / days they can't work) to remove days from this default.
const DEFAULT_HOURS = {
  weekday: { start: "16:30", end: "19:00" },  // Mon-Fri, after school
  weekend: { start: "10:00", end: "18:00" },  // Sat-Sun
};

const SUPABASE_URL = Deno.env.get("SUPABASE_URL")!;
const SERVICE_KEY = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
const REST = `${SUPABASE_URL}/rest/v1`;

const CORS = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "content-type",
  "Access-Control-Allow-Methods": "POST, OPTIONS",
};

function json(body: unknown, status = 200) {
  return new Response(JSON.stringify(body), {
    status, headers: { ...CORS, "Content-Type": "application/json" },
  });
}

function today() { return new Date().toISOString().split("T")[0]; }
function toMin(t: string) { const [h, m] = t.split(":").map(Number); return h * 60 + m; }
function pad(n: number) { return String(n).padStart(2, "0"); }
function fromMin(x: number) { return pad(Math.floor(x / 60)) + ":" + pad(x % 60); }
function addDays(iso: string, n: number) {
  const d = new Date(iso + "T00:00:00Z"); d.setUTCDate(d.getUTCDate() + n);
  return d.toISOString().split("T")[0];
}
function dowOf(iso: string) { return new Date(iso + "T00:00:00Z").getUTCDay(); } // 0 Sun .. 6 Sat

async function db(table: string, method: string, query = "", body?: unknown, prefer?: string) {
  const headers: Record<string, string> = {
    apikey: SERVICE_KEY, Authorization: `Bearer ${SERVICE_KEY}`, "Content-Type": "application/json",
  };
  if (prefer) headers["Prefer"] = prefer;
  const res = await fetch(`${REST}/${table}${query}`, {
    method, headers, body: body ? JSON.stringify(body) : undefined,
  });
  const text = await res.text();
  return { ok: res.ok, status: res.status, data: text ? JSON.parse(text) : null };
}

type Booking = { booking_date: string; booking_time: string; persons: string[] | null };
type TimeOff = { person: string; start_date: string; end_date: string };

// Is this person blocked (on holiday / off) on this date?
function isBlocked(person: string, iso: string, off: TimeOff[]) {
  return off.some(t => t.person === person && iso >= t.start_date && iso <= t.end_date);
}

// Build the implicit free-time ranges from the default hours, for every day in
// the booking horizon, skipping any day a person has booked off.
function buildRanges(off: TimeOff[]) {
  const ranges: { person: string; avail_date: string; start_time: string; end_time: string }[] = [];
  const start = today();
  for (let i = 0; i <= HORIZON_DAYS; i++) {
    const iso = addDays(start, i);
    const dow = dowOf(iso);
    const h = (dow === 0 || dow === 6) ? DEFAULT_HOURS.weekend : DEFAULT_HOURS.weekday;
    for (const person of PERSONS) {
      if (isBlocked(person, iso, off)) continue;
      ranges.push({ person, avail_date: iso, start_time: h.start, end_time: h.end });
    }
  }
  return ranges;
}

function computeFree(ranges: any[], bookings: Booking[]) {
  const out: { slot_date: string; slot_time: string; person: string }[] = [];
  for (const r of ranges) {
    const start = toMin(r.start_time), end = toMin(r.end_time);
    for (let t = start; t + JOB_MINUTES <= end; t += STEP_MINUTES) {
      const clashes = bookings.some(b => {
        if (b.booking_date !== r.avail_date) return false;
        const blocksThisPerson = b.persons === null || b.persons.includes(r.person);
        if (!blocksThisPerson) return false;
        const bt = toMin(b.booking_time);
        return t < bt + JOB_MINUTES && bt < t + JOB_MINUTES;
      });
      if (!clashes) out.push({ slot_date: r.avail_date, slot_time: fromMin(t), person: r.person });
    }
  }
  return out;
}

async function loadForCompute() {
  const [offR, booksR] = await Promise.all([
    db("time_off", "GET", `?end_date=gte.${today()}&order=start_date.asc`),
    db("bookings", "GET", `?booking_date=gte.${today()}&select=booking_date,booking_time,busy_persons`),
  ]);
  if (!offR.ok || !booksR.ok) return null;
  const bookings: Booking[] = (booksR.data || []).map((b: any) => ({
    booking_date: b.booking_date, booking_time: b.booking_time,
    persons: Array.isArray(b.busy_persons) && b.busy_persons.length ? b.busy_persons : null,
  }));
  return { ranges: buildRanges(offR.data || []), bookings };
}

Deno.serve(async (req: Request) => {
  if (req.method === "OPTIONS") return new Response("ok", { headers: CORS });
  if (req.method !== "POST") return json({ error: "Method not allowed" }, 405);

  let payload: any;
  try { payload = await req.json(); } catch { return json({ error: "Bad JSON" }, 400); }
  const action = payload?.action;

  if (action === "slots") {
    const ctx = await loadForCompute();
    if (!ctx) return json({ error: "Could not load times" }, 500);
    return json({ slots: computeFree(ctx.ranges, ctx.bookings) });
  }

  if (action === "create") {
    const b = payload.booking || {};
    const persons: string[] = Array.isArray(payload.persons) ? payload.persons : [];
    if (!b.name || !b.phone || !b.email || !b.booking_date || !b.booking_time) {
      return json({ error: "Missing required fields" }, 400);
    }
    if (persons.length) {
      const ctx = await loadForCompute();
      if (!ctx) return json({ error: "Could not check availability" }, 500);
      const free = computeFree(ctx.ranges, ctx.bookings);
      const allFree = persons.every(p =>
        free.some(f => f.slot_date === b.booking_date && f.slot_time === b.booking_time && f.person === p));
      if (!allFree) return json({ error: "taken" }, 409);
    }
    const row = {
      name: b.name, phone: b.phone, email: b.email,
      service_id: b.service_id ?? null, service_label: b.service_label ?? null,
      price: b.price ?? null, booking_date: b.booking_date, booking_time: b.booking_time,
      car: b.car ?? null, address: b.address ?? null, notes: b.notes ?? null,
      detailer: b.detailer ?? null, busy_persons: persons.length ? persons : null,
    };
    const r = await db("bookings", "POST", "", row, "return=minimal");
    return r.ok ? json({ ok: true }) : json({ error: "Could not save booking" }, 500);
  }

  if (payload?.password !== ADMIN_PASSWORD) return json({ error: "Unauthorized" }, 401);

  if (action === "list") {
    const r = await db("bookings", "GET", "?order=booking_date.asc,booking_time.asc");
    return r.ok ? json({ bookings: r.data }) : json({ error: "Could not load" }, 500);
  }
  if (action === "complete") {
    const r = await db("bookings", "PATCH", `?id=eq.${payload.id}`, { completed: !!payload.completed }, "return=minimal");
    return r.ok ? json({ ok: true }) : json({ error: "Could not update" }, 500);
  }
  if (action === "delete") {
    const r = await db("bookings", "DELETE", `?id=eq.${payload.id}`, undefined, "return=minimal");
    return r.ok ? json({ ok: true }) : json({ error: "Could not delete" }, 500);
  }

  // ----- Time off (holidays / days unavailable) -----
  if (action === "list_time_off") {
    const r = await db("time_off", "GET", `?end_date=gte.${today()}&order=start_date.asc,person.asc`);
    return r.ok ? json({ time_off: r.data }) : json({ error: "Could not load time off" }, 500);
  }
  if (action === "add_time_off") {
    const s = payload.time_off || {};
    if (!s.person || !s.start_date) return json({ error: "Missing time off fields" }, 400);
    const end = s.end_date || s.start_date;
    if (end < s.start_date) return json({ error: "End date must be on or after start date" }, 400);
    const people = s.person === "Both" ? PERSONS : [s.person];
    const rows = people.map(p => ({ person: p, start_date: s.start_date, end_date: end }));
    const r = await db("time_off", "POST", "", rows, "return=minimal");
    return r.ok ? json({ ok: true }) : json({ error: "Could not add time off" }, 500);
  }
  if (action === "delete_time_off") {
    const r = await db("time_off", "DELETE", `?id=eq.${payload.id}`, undefined, "return=minimal");
    return r.ok ? json({ ok: true }) : json({ error: "Could not delete time off" }, 500);
  }

  return json({ error: "Unknown action" }, 400);
});
```

---

## 8. What the owner can edit in `index.html`

Everything editable is near the top — search the file for **`EDIT`**:
- `businessName` (**"BB Car Detailing"**), `tagline`
  (**"Cleaner. Shinier. Better. …"**).
- `services[]` — `{id, label, price, blurb}`. Currently:
  Exterior Wash £25, Interior Only £25, Full Valet £40.
- `contacts[]` — currently **Bobby 07903 512940**, **Bramley 07434 651512**
  (each becomes a tappable "Call <name>" link in the footer).
- `facebookUrl` — set to the real page
  (`https://www.facebook.com/share/184bo5SHpP/?mibextid=wwXIfr`).
- `logoUrl` — `"logo.jpg"`. When set, the logo is **featured large in the hero
  only** (the H1 text is kept but visually hidden for SEO/screen readers). The
  sticky header and owner bar deliberately use **text branding** ("BB Car
  Detailing"), because the wide wordmark logo looked cramped/clipped shrunk into
  a small bar. Set to `""` to show the hero H1 text instead of the image.
- `gallery` — array of before/after pairs, e.g.
  `[{ before: "before1.jpg", after: "after1.jpg" }, …]`. Rendered as a
  **swipeable carousel** in the hero (dots appear when there's >1 pair). Empty
  array → friendly placeholders. Each filename must be an image uploaded/committed
  to the repo root.
- `apiUrl` — the Edge Function URL (do not change).
- Colour theme — the `:root` CSS variables.

---

## 9. Status

**Done (all merged to `main` and live; PRs #1–#9):**
- Landing page; **BB Car Detailing** branding + "Cleaner. Shinier. Better."
- **Logo** (`logo.jpg`) featured large in the hero; header + owner bar use clean
  **text branding** (the wide wordmark looked clipped in a small bar).
- **Facebook link** set to the real page.
- **Swipeable before/after carousel** built (config-driven via `CONFIG.gallery`;
  currently empty → placeholders until photos are added).
- Services & pricing; booking form → confirmation, saved to Supabase.
- **Booking flow:** customer picks a **date on a calendar** (native
  `<input type=date>`, constrained to the bookable window) then a **time** from a
  dropdown for that day. Date & Time fields are **stacked full-width** (a
  side-by-side row looked cramped on mobile). Native date input is CSS-normalised
  for iOS (see Gotchas).
- Password-protected owner page: bookings list (complete/delete).
- Two named contact numbers in footer.
- Availability system (**inverted model**): default hours every day
  (weekdays 16:30–19:00, weekends 10:00–18:00); owners book **time off** per
  person (single day or multi-day range, or "Both"); 30-min starts, 2½-hour
  overlap blocking, per-person + "together". Booking horizon **6 weeks**
  (`HORIZON_DAYS=42`), rolling.
- Owner page **colour-coded time-off calendar**: scrollable, shows current month
  + 6 months ahead (rolls forward automatically). Day colours: **blue=Bobby off,
  red=Bramley off, purple=both off**, normal=both free. Built from the same
  `time_off` data; chronological list with Remove buttons sits below it.
- **Marketing:** booking **QR code** saved as `bb-qr.png` (links to `…/#book`);
  a poster/leaflet prompt for "Claude design" saved in `POSTER-PROMPT.md`.

**TODO (waiting on the user to provide assets):**
1. **Before/after photos** — the user will send images (attached in chat → copy
   them from the session upload folder into the repo root). Add a
   `{ before, after }` line per pair to `CONFIG.gallery`; they appear in the hero
   carousel automatically. Merge to `main` to go live.
2. (Optional) the user is making a **poster** in Claude design using
   `POSTER-PROMPT.md`, their logo, the QR, and a photo of Bobby & Bramley — may
   ask for tweaks to the prompt or a Facebook caption.

**Good to know:**
- When a booking is **deleted** in the owner page, its time **reopens
  automatically** (availability is computed live on every `slots` request).
- The customer booking window (6 weeks) and the owner time-off calendar (6
  months) are **separate** ranges — don't confuse them.

---

## 10. Gotchas / notes

- **`index.html` is the single source of truth** for the site. An older
  `localStorage`-only file (`bb-details.html`) was removed — don't resurrect it.
- All times are **"HH:MM" strings** throughout (DB, function, site).
- Don't make `my-first-project` public or modify its non-BB files — it's an
  unrelated app; BB Details belongs in `Booking-system`.
- Supabase MCP tools are available via the same account. Use project id
  `qqkxxywdlqqkipioycaq`. `list_tables` / `get_logs` / `get_advisors` help when
  debugging.
- The network sandbox in these sessions blocks outbound calls to Supabase, so
  the function can't be `curl`-tested from the tool environment — test the DB
  layer with `execute_sql` and trust the function, or test live in the browser.
- **Native date input quirk:** iOS Safari renders `<input type=date>` wider,
  taller and centre-aligned. There's CSS normalising it
  (`-webkit-appearance:none`, left-aligned `::-webkit-date-and-time-value`) so it
  matches the other fields — keep it if you touch that area.
- **QR code:** `bb-qr.png` (repo root) links to
  `https://bobbyparnell0109.github.io/Booking-system/#book`. It's **black
  modules on a silver tile** (dark-on-light). Do NOT make it light-on-dark
  ("inverted") — those fail many scanners (verified failing to decode with
  OpenCV). Regenerate with the Python `qrcode` lib (`pip install qrcode pillow`)
  if the URL ever changes.
- Quick JS sanity check after editing `index.html`: extract the `<script>`
  blocks and `new vm.Script(s)` each in Node to catch syntax errors (there's no
  build step or test suite).

---

## 11. How to resume in a new session

1. Start the session on the **`Booking-system`** repo (add it to the session if
   prompted) so Claude can push directly and open/merge PRs.
2. First message: *"Read HANDOFF.md and continue the BB Car Detailing project."*
3. If scoped to the repo: develop on a feature branch, commit, push, then (when
   the user says "go live") open + squash-merge a PR into `main`, and resync the
   branch (see Section 3). If NOT scoped: edit `index.html` locally and send it
   to the user to upload via Safari.
4. The most likely next task is **before/after photos** (drop them into
   `CONFIG.gallery`) — see Section 9 TODO. The site is otherwise complete and
   live.
