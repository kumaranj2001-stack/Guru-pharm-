# Netlify Deployment Runbook — Guru Pharm

This repo deploys as a static React SPA, with Supabase as the backend.

---

## 1. Prerequisites in the repo

| Requirement | Notes |
|---|---|
| `package.json` with a `build` script | `npm run build` must produce the publish folder |
| `netlify.toml` | Already added — sets build command, publish dir, SPA redirect |
| `.nvmrc` | Already added — pins Node 20 so builds match local |

Publish folder by stack:

| Stack | Build command | Publish dir |
|---|---|---|
| **Vite** (default here) | `npm run build` | `dist` |
| Create React App | `npm run build` | `build` |
| Next.js | `next build` | `.next` — use `@netlify/plugin-nextjs`, drop the SPA redirect |

---

## 2. Connect the site

1. Netlify → **Add new site** → **Import an existing project**
2. Choose **GitHub** → `kumaranj2001-stack/Guru-pharm-`
3. Branch to deploy: **`main`**
4. Netlify reads `netlify.toml`, so the build settings should already be correct.
   Only fill them in manually if the fields come up blank.
5. **Deploy site**

Every push to `main` then redeploys automatically.

---

## 3. Environment variables (Supabase) — required

Netlify → **Site configuration → Environment variables → Add a variable**
(netlify.toml and `.env` are *not* used for secrets):

```
VITE_SUPABASE_URL      = https://<your-project-ref>.supabase.co
VITE_SUPABASE_ANON_KEY = <your anon/public key>
```

Rules that trip people up:

- **The `VITE_` prefix is mandatory.** Vite only exposes variables that start with
  `VITE_`, and only to code that reads `import.meta.env.VITE_SUPABASE_URL` /
  `import.meta.env.VITE_SUPABASE_ANON_KEY`.
- Vite **inlines these at build time**. After changing or adding one you must
  **Deploys → Trigger deploy → Clear cache and deploy site**. A plain "retry
  deploy" reuses the cached build and keeps the old values.
- Never put the Supabase **service_role** key in a frontend env var — it would be
  shipped to every visitor's browser. That key belongs only in server-side code.
- Lock down who can read data with Supabase **RLS policies**, since the anon key
  is public by design.

---

## 4. Point Supabase at the deployed URL

Supabase → **Authentication → URL Configuration**:

- **Site URL** → your Netlify URL, e.g. `https://<site-name>.netlify.app`
- **Redirect URLs** → add `https://<site-name>.netlify.app/**` *and*
  `http://localhost:5173/**` for local dev

Skip this and login / email-confirmation links will bounce back to `localhost`.

---

## 5. Common failures → fix

| Symptom | Cause | Fix |
|---|---|---|
| **404 on refresh or direct link** (e.g. `/settings`) | SPA fallback missing | The `[[redirects]]` block in `netlify.toml` and `public/_redirects` both fix it. Confirm they are the last rules and use `200`. |
| **Blank white page after deploy** | `base` path wrong, or stale `index.html` cached | Check build log for the Vite `base` option; caching headers in `netlify.toml` prevent the stale-shell case. |
| **Build fails: unsupported Node / syntax error** | Netlify used a Node version older than your local one | `.nvmrc` (Node 20) + `NODE_VERSION` in `netlify.toml` are already set. |
| **Build fails on `npm ci`** | `package-lock.json` out of sync | Commit the updated lockfile, or switch the command to `npm install`. |
| **App loads but Supabase calls fail / "supabaseUrl is required"** | Env var missing, misnamed, or not present at build time | Re-check the `VITE_` prefix, then **Clear cache and deploy**. |
| **Login redirects to localhost** | Supabase Site URL still points at localhost | Update Supabase **Site URL** (section 4). |
| **Works locally, 404s in prod on API-ish paths** | A catch-all `/api/*` proxy was assumed | There is no server here — all data comes from Supabase over HTTPS. |

---

## 6. The 404-on-refresh fix ships twice on purpose

Two files carry the same SPA fallback, and both are needed depending on how the
site is deployed:

| File | Used when |
|---|---|
| `netlify.toml` → `[[redirects]]` | Netlify builds the site from Git (the normal setup) |
| `public/_redirects` | You upload the build folder manually, **or** the site's build settings were overridden in the Netlify UI — in both cases `netlify.toml` is ignored, but everything in `public/` is copied into the build output, so this rule still ships |

If you update one, update the other so they stay identical. Both must end with
`/*    /index.html   200` — that `200` matters. A `301`/`302` makes the browser
change the URL and the router reports a wrong path instead of rendering the page.

**After the redirect is in place, redeploy and test by opening a deep link
directly** (e.g. paste `https://<site>.netlify.app/settings` into a fresh tab) —
navigating inside the app alone will not reproduce the original 404.

---

## 7. Deploy checklist

- [ ] `npm run build` succeeds locally
- [ ] `package-lock.json` committed
- [ ] `netlify.toml`, `public/_redirects` and `.nvmrc` present
- [ ] `VITE_SUPABASE_URL` + `VITE_SUPABASE_ANON_KEY` set in Netlify
- [ ] Netlify **Clear cache and deploy** run *after* env vars were added
- [ ] Supabase Site URL + Redirect URLs updated to the Netlify domain
- [ ] Deep link opened directly in a fresh tab to confirm no 404
- [ ] Browser hard-refresh after deploy confirms no stale shell
