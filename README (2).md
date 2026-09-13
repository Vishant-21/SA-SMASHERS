# SchoolShield

**Private, anonymous safety reporting for students — with a live dashboard for staff.**

SchoolShield lets a student report bullying, harassment, or an unsafe location in under a minute, without creating an account or giving their name. Each report gets a tracking code so the student can check on it later, and a college's staff can review, tag, and reply to reports from a shared dashboard.

This is a single self-contained `index.html` file — no build step, no framework tooling. Open it in a browser and it runs.

---

## Features

**Student side**
- Report a concern in three short steps: type → details → privacy
- Pick a state, then a college in that state (currently Goa, with ~65 colleges listed)
- Optional photo attachment
- Anonymous by default; students can optionally leave contact info for follow-up
- A tracking code (e.g. `SS-Q4453`) lets a student check status later — this code is the *only* way to look up a report, nothing is tied to an account
- Built-in safety tips page and an always-visible "Get help now" button with helpline numbers
- Two-way messaging: students can reply to staff follow-ups using their tracking code

**Staff side**
- State → college picker gates access to that college's reports only
- Lightweight passcode per college (see [Test credentials](#test-credentials) below)
- Overview tab: report volume trend, breakdown by type and severity, quick stats
- All-reports tab: filterable table, click into any report for full details, photo, and reply thread
- Status tracking per report: Received → Under review → Action taken → Resolved
- Reports are auto-tagged with a rough severity (low/medium/high) based on keywords in the description — see [How severity tagging works](#how-severity-tagging-works)

---

## Tech stack

- **React 18** (via CDN, no build step) — the whole UI is `React.createElement` calls, no JSX/webpack
- **Supabase** (Postgres) — shared backend so reports sync across every device, not just one browser
- Plain CSS (no framework), Inter/Sora via Google Fonts

## How it's wired together

Everything reads and writes through one `storage` object with four methods: `get`, `set`, `delete`, `list`. Under the hood, that object talks to a single Supabase table:

```sql
create table kv_store (
  key text primary key,
  value text not null,
  updated_at timestamptz default now()
);
```

Reports, staff passcodes, and everything else are stored as JSON strings under namespaced keys (e.g. `schoolshield:report: SS-Q4453`). Keeping this key/value shape means the storage backend can be swapped again later (e.g. for a proper multi-tenant database) without touching the rest of the app.

### Row Level Security

The Supabase table currently uses a permissive demo policy:

```sql
alter table kv_store enable row level security;

create policy "public read/write for demo"
  on kv_store for all
  using (true)
  with check (true);
```

This means **anyone with the app's Supabase key can read or overwrite any row** — fine for a hackathon demo with fake data, but it should be tightened (or replaced with real auth-scoped policies) before any real student data goes anywhere near it.

---

## Running it

There's nothing to install. Either:

- Open `index.html` directly in a browser, or
- Serve it with any static file host (Netlify, Vercel, GitHub Pages — drag-and-drop deploy works fine since it's one file)

The Supabase project URL and public key are already set near the top of the script:

```js
const SUPABASE_URL = "https://huesnoylplzisrwyxfnq.supabase.co";
const SUPABASE_ANON_KEY = "sb_publishable_...";
```

The key here is a **publishable** key — it's designed to be exposed client-side; access control is enforced by the RLS policy on the table, not by keeping this key secret. To point the app at a different Supabase project, replace both values and re-run the `create table` + policy SQL above against the new project.

---

## Test credentials

For demo/judging purposes, two colleges have staff access codes already set up:

| College | Access code |
|---|---|
| Padre Conceicao College of Engineering | `12345678` |
| Goa College of Engineering | `24242424` |

To try the staff dashboard: from the home screen, choose **Staff dashboard** → state **Goa** → pick one of the two colleges above → enter its code.

---

## How severity tagging works

Reports are auto-tagged using simple keyword matching, **not** machine learning:

- Certain words (weapon, threat, hit, choke, etc.) mark a report **high** severity
- Words suggesting a pattern ("every day", "repeatedly", "for weeks") bump severity to at least **medium**
- Words suggesting the incident happened online tag the report as "online"
- "Unsafe location" reports default to at least **medium** severity

This is a heuristic meant to help staff triage faster, not a clinical or definitive risk assessment — worth being upfront about if asked how the "AI" works, since there isn't a model involved.

---

## Known limitations

- **RLS policy is wide open** (see above) — acceptable for a demo, not for production data.
- **Staff passcodes are not real authentication** — they're a plaintext string comparison stored in the same shared table, not a hashed credential or session system.
- **Severity tagging is keyword-based**, not NLP/ML — it can be fooled or miss context.
- **No rate limiting** — nothing currently stops spam report submissions.

## Future scope

- Native mobile app (iOS/Android) for faster, more private access
- ML-based severity triage to replace the current keyword heuristic
- Multilingual support for regional languages
- Real-time push notifications for staff on high-severity reports

---

Built for a hackathon submission — see the accompanying slide deck for the full pitch, problem statement, and feasibility notes.

