# CLAUDE.md — PRIME Coach

> Project memory + architecture contract for Claude Code / Claude sessions.
> Read this first. Keep it current. When something here is wrong, fix the doc in the same change.

---

## 1. What this is

**PRIME Coach** is a warm-up, mobility, activation, and recovery tool for a personal trainer.
The trainer serves: weight-training clients, basketball players, general-fitness adults, youth
athletes, and adults with tight hips / limited mobility.

The product grows in two phases. **Build Phase 1 so it upgrades into Phase 2 without a rewrite.**
The single decision that makes that possible is the **Store seam** (§4). Everything reads and
writes through `Store`. The UI never touches storage directly. Swap what's *behind* `Store` and the
app moves from offline-only to cloud-synced without touching view code.

- **Phase 1 — Coach's offline tool (current).** One user (the trainer). No accounts, no server,
  no cost. He manages a client list, builds per-client programs from an exercise library, and runs
  a client through a session with a full-screen guided timer. Installs to a phone/tablet home screen
  as a PWA and works fully offline. Data lives on-device (IndexedDB).
- **Phase 2 — Two-sided platform (later, when clients grow).** Accounts with **Coach** and
  **Client** roles linked by an invite code. Clients build/propose their own programs; the coach
  receives them in a dashboard, reviews/approves, and coaches sessions either **async** (client runs
  solo, logs completion + pain flags, coach reviews after) or **live** (realtime step sync). Ships
  as a responsive web app for both roles first; native app-store build comes after via Capacitor when
  push notifications/reminders matter.

**Non-goals (keep scope honest):** not a medical device, not a full periodization/strength-program
platform (it's warm-up/recovery + session running), no built-in video calling in early Phase 2 (link
out to Zoom/Daily; integrate LiveKit/Daily only if live coaching demand is real), no payments/billing
until there's a paying base. Compare against TrueCoach / Trainerize / TrainHeroic for "buy vs build"
gut-checks — this is deliberately a focused, coach-styled subset.

---

## 2. Tech stack

### Phase 1 (current) — zero-build static PWA
Chosen for **zero friction**: the trainer (and later, non-technical clients) need no toolchain, and
it hosts as plain static files anywhere.

- Single `index.html` — vanilla JS, no framework, no build step. Hash-free in-memory view router.
- `Store` data layer over **IndexedDB**, with an **in-memory fallback** when IDB is unavailable
  (e.g. sandboxed preview, private mode). Async API from day one so the contract already matches a
  network-backed implementation.
- PWA: `manifest.webmanifest` + `sw.js` (cache-first app shell, runtime-cache fonts) + icons.
- Fonts: Bebas Neue + Inter via Google Fonts CDN, **with system fallbacks** so the app degrades
  gracefully offline. SW caches them after first load.

### Phase 2 (target) — same product, cloud + roles
- **React + TypeScript + Vite.** Reuse all Phase 1 domain content (exercise library, routines,
  safety copy) and port the runner logic to a `<Runner>` component.
- **Dexie** (typed IndexedDB) behind `Store` for offline cache; **Supabase** (Postgres + Auth +
  Row Level Security + Realtime) as the cloud backend behind the *same* `Store` interface. Firebase is
  the fallback option; prefer Supabase for relational data + RLS.
- `vite-plugin-pwa` for the service worker / offline.
- **Capacitor** to wrap the same web app for iOS/Android app stores (Phase 2b). **Tauri** if a desktop
  build is ever wanted.
- Live session sync: Supabase Realtime (broadcast/presence on a `session` row). Video: link out
  first, then Daily or LiveKit.

> Rule of thumb for the jump: the **view layer** is rebuilt in React; the **data models (§3)**,
> the **`Store` contract (§4)**, and the **domain content (§6)** carry across unchanged. That's the
> whole point of the seam.

---

## 3. Data models (canonical — cloud-shaped from day one)

Model everything now in the shape it will have in Postgres later. Every record carries a UUID `id`,
an `ownerId` (the coach; unused in Phase 1, set to `"local"`, becomes `auth.uid()` in Phase 2), and
`createdAt` / `updatedAt` ISO timestamps. IDs are `crypto.randomUUID()` — collision-free offline and a
clean Postgres `uuid` primary key later.

```ts
type ID = string; // crypto.randomUUID()

type Client = {
  id: ID;
  ownerId: ID;            // coach; "local" in Phase 1
  name: string;
  level: 'Beginner' | 'Intermediate' | 'Athlete';
  sport: string;          // free text e.g. "Basketball — guard", "General fitness"
  goals: string;          // free text
  injuries: string;       // free text; surfaced on the client + links to pain rules
  notes: string;          // free text
  createdAt: string;
  updatedAt: string;
};

type Step = {             // one move inside a program
  name: string;
  dose: string;           // "30 sec" | "10/side" | "2x20 m" — parseSeconds() reads time vs reps
  cue: string;            // the coaching cue shown front-and-center in the runner
};

type Program = {
  id: ID;
  ownerId: ID;
  clientId: ID;           // FK -> Client.id
  title: string;
  kind: 'warm' | 'cool';  // drives runner color theme (warm = amber, cool = teal) and tab grouping
  source: 'custom' | 'template'; // template = seeded from a built-in routine, then editable
  steps: Step[];
  createdAt: string;
  updatedAt: string;
};

// Phase 2 additions (do not build in Phase 1, but reserve the shape):
type SessionLog = {
  id: ID;
  programId: ID;
  clientId: ID;
  completedAt: string;
  durationSec: number;
  painFlags: { step: string; score: number; note?: string }[];
  notes: string;
};
type CoachClientLink = { id: ID; coachId: ID; clientId: ID; status: 'invited'|'active'; inviteCode: string };
```

**Postgres mapping (Phase 2):** `clients`, `programs` (jsonb `steps` or a normalized `program_steps`
table — start jsonb, normalize only if querying inside steps becomes necessary), `session_logs`,
`coach_client_links`. RLS: a coach sees rows where `ownerId = auth.uid()`; a client sees their own
`clients` row, their `programs`, and their `session_logs`.

---

## 4. The Store seam (the most important thing in this repo)

All persistence goes through one async object. **No view, no handler, no component reads or writes
storage directly.** This is the line Phase 2 swaps under.

```ts
interface Store {
  init(): Promise<void>;

  listClients(): Promise<Client[]>;
  getClient(id: ID): Promise<Client | null>;
  saveClient(c: Client): Promise<Client>;      // upsert by id
  deleteClient(id: ID): Promise<void>;          // also cascades that client's programs

  listPrograms(clientId?: ID): Promise<Program[]>; // all, or filtered by client
  getProgram(id: ID): Promise<Program | null>;
  saveProgram(p: Program): Promise<Program>;    // upsert by id
  deleteProgram(id: ID): Promise<void>;

  getMeta(key: string): Promise<any>;           // settings / schema version / last-used
  setMeta(key: string, value: any): Promise<void>;
}
```

- **Phase 1 impl:** IndexedDB (`prime` db; stores `clients`, `programs` w/ `clientId` index, `meta`),
  with an in-memory `Map` fallback if `indexedDB.open` throws or is missing. Same return shapes either
  way, so the preview demo works (ephemerally) and a real browser persists.
- **Phase 2 impl:** the same methods backed by Supabase queries (+ Dexie as an offline cache and an
  outbox for writes made offline). The interface does not change. Async-from-day-one means no call
  site has to change from sync to async during the migration.

**If you add a feature that needs to persist something new, add a method to `Store` — never reach into
IndexedDB (or later Supabase) from a view.**

---

## 5. Repo structure

### Phase 1 (actual)
```
prime-coach/
  index.html              # the whole app: styles, data, views, router, Store, runner, PWA reg
  manifest.webmanifest    # PWA manifest
  sw.js                   # service worker (app-shell cache + runtime font cache)
  icon-192.png            # PWA / home-screen icon
  icon-512.png            # PWA / splash icon (maskable-safe padding)
  README.md               # how to run offline, host, and install
  CLAUDE.md               # this file
```

### Phase 2 (target sketch — create when starting the rebuild)
```
src/
  data/
    models.ts             # the types in §3 (source of truth)
    store.ts              # Store interface (§4)
    store.local.ts        # Dexie/IndexedDB implementation
    store.cloud.ts        # Supabase implementation
    content/              # ported domain content (§6): exercises.ts, routines.ts, safety.ts
  features/
    clients/              # list, detail, form
    programs/             # builder, exercise picker
    runner/               # <Runner> ported from Phase 1 runner logic
    sessions/             # async logging + live sync (Phase 2)
    auth/                 # login, role gate, invite flow (Phase 2)
  app/                    # router, shell, tab nav, theme tokens
```

---

## 6. Domain content inventory (what already exists — don't recreate, reuse)

All of this lives in `index.html` today and is the thing to port into `src/data/content/` in Phase 2.

- **Exercise library** (`EX` array, ~60 moves). Each: `id, n(ame), cat, dose, areas[], cue, miss
  (mistake to avoid), pro(gression), reg(ression)`, plus `target` on activation moves.
  - `cat`: `dyn` (dynamic warm-up) · `act` (activation) · `sta` (static/cooldown) · `plyo`
    (landing/plyo). This drives the colored badge and the runner theme.
- **Built-in routines** (`R` object, 8 routines): `q5, w10, a15, c5, c10, bball, lift, recovery`.
  Each: `title, mins, tag(warm|cool), level, type, color, blurb, scale{Beginner,Intermediate,Athlete},
  steps[[name,dose,cue], ...]`. In Phase 1 these double as **templates** — cloning one creates an
  editable `Program` for a client (`source:'template'`).
- **Mobility areas** (`MOB_AREAS`): Hips, Hamstrings, Groin/Adductors, Ankles, Calves, Lats,
  Shoulders, T-Spine. **Activation targets** (`ACT_TARGETS`): Glutes, Core, Adductors, Calves,
  Stabilizers.
- **Safety + coaching content** (static views): traffic-light **pain rules** + modification ladder,
  **red-flag / refer-out** list, **client intake** questions, pre/post/youth **coach scripts**,
  **weekly use** guide, **printable** trainer cards (`@media print` + `window.print()`).

---

## 7. The runner (signature feature)

Full-screen guided session player. Shot-clock countdown for time-based steps (`"30 sec"`, `"2 min"`),
count-up for rep-based steps (`"10/side"`) with a manual Next. Shows the coach cue large, progress
dots, prev/pause/next, an auto-advance toggle, and a WebAudio beep at step end.

**Contract:** `startRunner(routine)` takes a **routine object** `{ title, tag, steps:[{...}|[name,dose,cue]] }`
— NOT a key into `R`. Built-in routines call it via `runBuiltin(key)` → `startRunner(R[key])`; saved
programs call `runProgram(id)` → load from `Store` → `startRunner(programAsRoutine)`. Keeping the runner
decoupled from the hardcoded `R` table is deliberate — it's what lets it run user-built programs and
what makes the Phase 2 `<Runner>` port a straight lift. `parseSeconds(dose)` is the time-vs-reps
detector; keep its accepted formats in sync with the dose strings used in content.

---

## 8. Conventions & gotchas

- **Doses are strings, parsed at runtime.** Time formats the runner understands: `N sec`, `N min`,
  `Ns`. Anything else (`10/side`, `2x20 m`, `15 reps`) is treated as rep-based (count-up, manual
  advance). If you introduce a new dose format, update `parseSeconds()` or it silently becomes
  rep-based.
- **Color = meaning, not decoration.** Amber = warm-up/raise, volt-green = activation, teal = cool/
  recovery/mobility, steel-blue = weights/landing, red = pain/red-flags. `kind:'warm'|'cool'` on a
  program selects the runner theme. Don't reassign these.
- **IndexedDB from `file://` is unreliable** across browsers, and **service workers won't register off
  `file://` at all** (they need `https://` or `localhost`). So "real" offline + install + persistence
  requires hosting the files once (one drag-and-drop to Netlify/Vercel/Cloudflare Pages, or a local
  static server). Opening `index.html` directly is fine for a quick UI look but is not the install path.
  See README.
- **Preview sandboxes** (and private mode) may block IndexedDB — that's why `Store` has the in-memory
  fallback. Don't remove it; it's what keeps the in-app preview demo working.
- **No `localStorage`/`sessionStorage`.** Use `Store` (IndexedDB/in-memory). Browser storage APIs are
  blocked in some embedded preview contexts; `Store` centralizes and guards all of it.
- **Views are string-builders + an `afterRender` hook.** Synchronous `VIEWS[name]()` returns HTML;
  data-driven screens (clients, client detail, builder) return a skeleton and an async loader fills it
  in after mount. Follow that pattern rather than making the router async.
- **Builder state** is a single module-global object rebuilt by `renderBuilder()`. Persist on explicit
  Save and on Run (autosave-on-edit is a no-op in the preview anyway).
- **Disclaimer stays.** This is a coaching tool, not medical advice; the red-flag/refer-out content and
  the footer line are load-bearing, not boilerplate. Keep them.
- **Branding is placeholder.** "PRIME" is a working name; the app is meant to be rebranded for the
  trainer's studio. Keep brand strings centralized enough to swap quickly.

---

## 9. Phase 2 migration playbook (when clients grow)

1. **Scaffold** `Vite + React + TS`. Port §6 content into `src/data/content/*` (pure data, no DOM).
2. **Lift the models (§3) verbatim** into `models.ts`. Define the `Store` interface (§4).
3. **Port the runner** to `<Runner routine={...} />` — same state machine, React state instead of the
   `run` global.
4. **Rebuild views** as components (clients, detail, form, builder, picker) against `Store` — start on
   `store.local.ts` (Dexie) so the offline app reaches parity with Phase 1.
5. **Add Supabase**: project, `clients`/`programs`/`session_logs`/`coach_client_links` tables, Auth,
   and RLS (coach sees `ownerId = auth.uid()`; client sees own rows). Implement `store.cloud.ts`
   against the *same* interface. Add a Dexie outbox so offline writes sync on reconnect.
6. **Roles + invite flow**: coach generates an invite code → client signs up → `coach_client_links`
   row → client sees their assigned programs; coach sees the client's submitted programs + logs.
7. **Sessions**: async first (client runs `<Runner>`, writes a `SessionLog`, coach reviews). Then live
   (Supabase Realtime broadcast on a `session` row; coach advances steps, client screen follows).
8. **Native** (2b): wrap with Capacitor for iOS/Android; add push for session reminders. Consider Daily/
   LiveKit only if live *video* is actually requested.

Each step is shippable on its own. Don't build 5–8 before 1–4 are solid and in real use.

---

## 10. Status / changelog

- **Phase 0 (done):** Reference + runner prototype — exercise library, 8 routines, safety content,
  guided timer, print mode, "find my routine" wizard. (`prime-warmup-system.html`)
- **Phase 1 (current):** Adds `Store` (IndexedDB + in-memory fallback), Clients (CRUD), per-client
  Program Builder (library picker, inline dose/cue edit, reorder, template clone), runner refactor to
  run saved programs, and the PWA shell (manifest + service worker + icons) for installable offline use.
- **Next:** see §9. First Phase 2 milestone = scaffold React/TS + port content + reach offline parity.
