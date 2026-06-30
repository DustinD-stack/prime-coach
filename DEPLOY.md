# Deploy PRIME Coach to GitHub Pages

GitHub Pages is free, serves over HTTPS (required for the PWA to install, run offline, and save
data), and this app uses **all-relative paths**, so it works from a project URL like
`https://YOURNAME.github.io/prime-coach/` with no code changes.

> First unzip `prime-coach.zip` so you have the loose files. **`index.html` must end up at the top
> level of the repository** — not inside a nested folder.

---

## Method A — GitHub website (no terminal)

Works from a computer browser. (On a phone the GitHub app sends you to the browser for these steps
anyway, so just use Safari/Chrome.)

1. **Create the repo.** On github.com, click **+ → New repository**. Name it `prime-coach`, set it
   **Public** (free Pages needs a public repo), and click **Create repository**.
2. **Upload the files.** On the empty repo page, click **Add file → Upload files**. Drag in all the
   files from the `prime-coach` folder:
   `index.html`, `sw.js`, `manifest.webmanifest`, `icon-192.png`, `icon-512.png`, `README.md`,
   `CLAUDE.md` (and this `DEPLOY.md`). Confirm `index.html` is at the root, then **Commit changes**.
3. **Turn on Pages.** Go to **Settings → Pages**. Under **Build and deployment → Source**, choose
   **Deploy from a branch**. Set branch to **`main`** and folder to **`/ (root)`**. Click **Save**.
4. **Open your site.** Wait ~1 minute, refresh the Pages settings page, and use the URL it shows:
   `https://YOURNAME.github.io/prime-coach/`.
5. **Install on your phone.** Open that URL on the device → **Add to Home Screen** (iOS Safari:
   Share → Add to Home Screen; Android Chrome: menu → Install app).

---

## Method B — git command line (from a computer)

```bash
cd path/to/prime-coach          # the folder with index.html in it
git init
git add .
git commit -m "PRIME Coach — initial deploy"
git branch -M main
git remote add origin https://github.com/YOURNAME/prime-coach.git
git push -u origin main
```

Then do **step 3** above (Settings → Pages → Deploy from a branch → `main` / root → Save).

---

## Updating the site later

1. Edit files (e.g. `index.html`) and re-upload / push them.
2. **Bump the cache** so installed copies pull the new version: open `sw.js` and change
   `const CACHE = 'prime-coach-v1';` to `v2`, `v3`, etc. Without this, the service worker keeps
   serving the previously cached build.

---

## Good to know

- **Public repo, private data.** Free Pages requires the repo be public, but that only exposes the
  *code* — client and program data never leaves each device (it lives in the browser, not the repo).
- **Relative paths are already set** (`.`, `sw.js`, `icon-192.png`), which is why the app works in a
  `/prime-coach/` subpath. Don't change them to absolute `/...` paths or it will break on Pages.
- **`.nojekyll` (optional).** None of these files start with `_`, so it isn't required. If you ever
  add such a file, drop an empty `.nojekyll` at the repo root to skip GitHub's Jekyll processing.
- **Custom domain (optional).** Settings → Pages → Custom domain lets you point your own domain at it
  later; the install/offline behavior is identical.
