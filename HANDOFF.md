# BB Details — Project Handoff & Context

> **Read this first.** This file is the complete state of the BB Details
> project. A new Claude Code session can read it and continue at full
> capability. Last updated: 2026-06-17.

---

## 1. What this project is

A single-page **mobile car valeting website** for **BB Details**.
Owners: **Bobby** and **Bramley**. Dark theme (blacks/greys), mobile-first
(customers arrive from Facebook/Instagram links).

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

**If a future session IS scoped to Booking-system**, you can push `index.html`
directly with the GitHub tools and skip the manual upload — much better.
Consider asking the user to add Booking-system to the session.

Backend changes (Supabase) are done by Claude directly via the Supabase MCP
tools — no user action needed.

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
- `ADMIN_PASSWORD = "bbdetails2026"`.
- To change any of these: edit the function source (Section 7) and redeploy
  via the Supabase MCP `deploy_edge_function` (keep `verify_jwt = false`).

### How availability works
Owners add free-time **ranges** (person, date, from, until). The function
generates start times every 30 min where `start + 150min <= end`, then removes
any start whose 2½-hour job would **overlap** an existing booking for that
person (overlap test: `t < bookingTime+150 && bookingTime < t+150`). So booking
11:00 (job to 13:30) on a 09:00–17:00 day leaves only 13:30, 14:00, 14:30.
If **both** people are free at the same start time, the website combines them
into a single **"Bobby & Bramley (together)"** option; booking it ties up both.

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

-- Availability ranges
create table public.availability (
  id          uuid primary key default gen_random_uuid(),
  created_at  timestamptz not null default now(),
  person      text not null,
  avail_date  date not null,
  start_time  text not null,          -- "HH:MM"
  end_time    text not null,          -- "HH:MM"
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
- `{ action: "list_ranges" }` → upcoming availability ranges
- `{ action: "add_range", range: {person, avail_date, start_time, end_time} }`
- `{ action: "delete_range", id }`

---

## 7. Edge Function source (`bb-bookings`, current deployed version)

> If you change this, redeploy with the Supabase MCP `deploy_edge_function`
> (project `qqkxxywdlqqkipioycaq`, name `bb-bookings`, `verify_jwt: false`).

```ts
import "jsr:@supabase/functions-js/edge-runtime.d.ts";

const ADMIN_PASSWORD = "bbdetails2026";  // <-- change me
const JOB_MINUTES = 150;                 // length of one valet (2.5 hours)
const STEP_MINUTES = 30;                 // offer a start time every 30 mins

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
  const [rangesR, booksR] = await Promise.all([
    db("availability", "GET", `?avail_date=gte.${today()}&order=avail_date.asc,start_time.asc`),
    db("bookings", "GET", `?booking_date=gte.${today()}&select=booking_date,booking_time,busy_persons`),
  ]);
  if (!rangesR.ok || !booksR.ok) return null;
  const bookings: Booking[] = (booksR.data || []).map((b: any) => ({
    booking_date: b.booking_date, booking_time: b.booking_time,
    persons: Array.isArray(b.busy_persons) && b.busy_persons.length ? b.busy_persons : null,
  }));
  return { ranges: rangesR.data || [], bookings };
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
  if (action === "list_ranges") {
    const r = await db("availability", "GET", `?avail_date=gte.${today()}&order=avail_date.asc,start_time.asc,person.asc`);
    return r.ok ? json({ ranges: r.data }) : json({ error: "Could not load ranges" }, 500);
  }
  if (action === "add_range") {
    const s = payload.range || {};
    if (!s.person || !s.avail_date || !s.start_time || !s.end_time) return json({ error: "Missing range fields" }, 400);
    if (toMin(s.end_time) <= toMin(s.start_time)) return json({ error: "End time must be after start time" }, 400);
    const r = await db("availability", "POST", "",
      { person: s.person, avail_date: s.avail_date, start_time: s.start_time, end_time: s.end_time }, "return=minimal");
    if (r.ok || r.status === 409) return json({ ok: true });
    return json({ error: "Could not add range" }, 500);
  }
  if (action === "delete_range") {
    const r = await db("availability", "DELETE", `?id=eq.${payload.id}`, undefined, "return=minimal");
    return r.ok ? json({ ok: true }) : json({ error: "Could not delete range" }, 500);
  }

  return json({ error: "Unknown action" }, 400);
});
```

---

## 8. What the owner can edit in `index.html`

Everything editable is near the top — search the file for **`EDIT`**:
- `businessName`, `tagline`
- `services[]` — `{id, label, price, blurb}`. Currently:
  Exterior Wash £25, Interior Only £25, Full Valet £40.
- `contacts[]` — currently **Bobby 07903 512940**, **Bramley 07434 651512**
  (each becomes a tappable "Call <name>" link in the footer).
- `facebookUrl` — currently the placeholder `https://www.facebook.com/`.
- `apiUrl` — the Edge Function URL (do not change).
- Colour theme — the `:root` CSS variables.

---

## 9. Status

**Done:**
- Landing page (name, logo placeholder, tagline, before/after photo placeholders)
- Services & pricing
- Booking form → confirmation, saved to shared Supabase backend
- Password-protected owner page: bookings list (sorted by date, complete/delete)
- Two named contact numbers in footer
- Availability system: free-time ranges → 30-min start times, 2½-hour overlap
  blocking, per-person + "together"

**TODO (waiting on the user to provide assets):**
1. **Facebook link** — set `CONFIG.facebookUrl` to the real page URL.
2. **Logo** — replace the `<div class="logo">BB</div>` (appears twice: header
   + admin header) with `<img src="logo.png" …>`; the image file must also be
   uploaded to the Booking-system repo and referenced relatively.
3. **Before/after photos** — replace the two `<div class="photo">…</div>`
   placeholders in the hero `.gallery` with `<img>` tags; upload the image
   files to the repo too.

**Good to know:**
- When a booking is **deleted** in the owner page, its time **reopens
  automatically**. Availability is computed live from the *current* bookings on
  every `slots` request, so a removed booking no longer blocks anything. (This
  was a limitation of the earlier fixed-slot design; the range design fixed it.)

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

---

## 11. How to resume in a new session

1. Start the session on the **`Booking-system`** repo (add it to the session if
   prompted) so Claude can push `index.html` directly.
2. First message: *"Read HANDOFF.md and continue the BB Details project."*
3. Provide whatever's ready (Facebook URL / logo image / before+after photos)
   and Claude will wire it in, then either push (if scoped to the repo) or hand
   back the updated `index.html` to upload.
