# PRIME Coach — Phase 1 (offline app)

A warm-up, mobility, activation, and recovery tool for a personal trainer. Phase 1 is the **coach's
offline app**: manage a client list, build and save each client's programs from a built-in exercise
library, and run them through a session with a full-screen guided timer. It installs to a phone or
tablet home screen and works with no internet.

Everything is in `index.html` (no build step, no dependencies). The other files make it installable.

```
index.html              the whole app
manifest.webmanifest    makes it installable
sw.js                   service worker → offline + home-screen install
icon-192.png / 512.png  app icons
CLAUDE.md               architecture + the Phase 2 plan (read before extending)
README.md               this file
```

---

## Run it — 3 ways

### 1. Quick look (no install)
Double-click `index.html` to open it in a browser. You can click through everything and the builder
works. **Caveat:** opened this way (`file://`), the browser may not persist saved clients between
launches, and it can't install to the home screen. Use option 2 or 3 for the real thing.

### 2. Host it once (recommended — enables offline + install + saving)
Service workers and reliable storage need a real web address (`https://` or `localhost`). Easiest free
options, no account-heavy setup:

- **Netlify Drop** — go to https://app.netlify.com/drop and drag the whole `prime-coach` folder onto
  the page. You get a live URL in seconds. (Vercel and Cloudflare Pages work the same way.)
- **Or run a local server** from inside the folder:
  - Python: `python3 -m http.server 8080` → open `http://localhost:8080`
  - Node: `npx serve` → open the URL it prints

Once it's loaded from a URL, it's cached and keeps working offline.

### 3. Install to a phone / tablet
Open the hosted URL (from option 2) on the device:
- **iPhone/iPad (Safari):** Share → **Add to Home Screen**.
- **Android (Chrome):** menu → **Install app** / **Add to Home Screen**.

It then launches full-screen like a native app and runs offline.

---

## How it works

- **Clients tab** → add a client (name, level, sport/focus, goal, injury notes). Open a client to see
  their saved programs.
- **Build a program** → name it, pick Warm-Up or Cooldown, add moves from the library, set each move's
  dose and coach cue, reorder with the arrows. Or **Start from a template** to clone one of the eight
  built-in routines and edit it.
- **Run** any saved program → the full-screen timer counts down timed moves, lets you advance rep-based
  moves manually, shows the coaching cue big, and beeps between steps.
- The **Home, Warm-Up, Cooldown, and Tools** tabs keep the original reference library, the "find my
  routine" wizard, pain rules, red-flags, intake questions, coach scripts, and printable cards.

### Where the data lives
Clients and programs are stored **on the device, in that browser** (IndexedDB) — nothing is uploaded,
no account needed. Because it's per-device/per-browser, the data doesn't sync across devices yet.
That's exactly what Phase 2 adds (accounts + cloud sync + a client-facing side). See `CLAUDE.md`.

### Rebranding
"PRIME" is a placeholder. To restyle for a specific studio, change the brand text in the top bar and
footer of `index.html`, the colors in the `:root` CSS block, the `name`/`short_name` in
`manifest.webmanifest`, and swap the two icons. After any change to `index.html`, bump the `CACHE`
version string in `sw.js` so installed copies pick up the update.

---

## What's next (Phase 2)
Accounts with **Coach** and **Client** roles, clients building/submitting their own programs, the coach
reviewing and approving, and coaching sessions either async (client logs completion + pain flags) or
live (realtime step sync). Planned as a React + TypeScript rebuild on the same data models, backed by
Supabase, wrapped for the app stores with Capacitor. The full plan, data models, and migration steps
are in **`CLAUDE.md`**.

*PRIME Coach is a coaching tool, not medical advice. Keep the red-flag / refer-out guidance intact.*
