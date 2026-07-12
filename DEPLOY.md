# Deploying SyHire to Vercel via GitHub

This is a **static site** — no build step, no server. Vercel just serves the
files. The live page is `SyHire.standalone.html` (fully self-contained: all
CSS, JS, React, and the logo are inlined, so it works with zero external calls).

---

## 1. Push the project to GitHub

From this project folder on your machine:

```bash
git init
git add .
git commit -m "SyHire — initial deploy"
git branch -M main
git remote add origin https://github.com/<your-username>/syhire.git
git push -u origin main
```

Create the empty `syhire` repo on github.com first (no README/license — you
already have files). `.gitignore` keeps your private working files
(`design_handoff_syhire_v1/`, `screenshots/`, `uploads/`, `SECURITY_AUDIT.md`)
**out** of the public repo automatically.

## 2. Connect the repo to Vercel

1. Go to <https://vercel.com/new>.
2. **Import** your `syhire` GitHub repo (authorize GitHub if prompted).
3. Framework preset: **Other**. Build command: **leave empty**. Output
   directory: **leave empty** (it serves the repo root).
4. Click **Deploy**.

Vercel reads `vercel.json` automatically — security headers, caching, clean
URLs, and the `/` → `SyHire.standalone.html` rewrite all apply on deploy.

Every future `git push` to `main` redeploys automatically.

## 3. After it's live — swap the placeholder domain

The security/SEO config still points at the placeholder `syhire.app`. Once you
have your real domain, find-and-replace it in these files:

- `SyHire.standalone.html` (canonical URL + OG/Twitter meta)
- `SyHire.dc.html` (same meta, your editable source)
- `robots.txt`
- `sitemap.xml`

Then add the domain under **Vercel → Project → Settings → Domains**.

---

## Protecting your files — what's already in place

Your `vercel.json` ships a hardened header set on every response:

- **Content-Security-Policy** — locks what can load/run (no arbitrary scripts).
- **X-Frame-Options: DENY** + CSP `frame-ancestors 'none'` — no one can iframe
  your site (clickjacking + embedding protection).
- **Strict-Transport-Security** — forces HTTPS.
- **X-Content-Type-Options, Referrer-Policy, Permissions-Policy, COOP, CORP** —
  defense-in-depth hardening.
- Immutable caching scoped to `/assets/*` only.

`.gitignore` protects your **private working files** from ever reaching the
public GitHub repo.

### Want the site itself password-protected?

A public Vercel deploy is visible to anyone with the URL. To gate it:

- **Vercel → Project → Settings → Deployment Protection → Password Protection**
  (or Vercel Authentication) — requires a Pro/Enterprise plan.
- Or keep the GitHub repo **private** so the source code isn't public even
  though the deployed page is.

### Full-source privacy option

If you want to publish **only** the finished page and keep your editable
`.dc.html` source private, uncomment the last block in `.gitignore` — Vercel
still deploys fine from `SyHire.standalone.html` alone.

---

> **Honest note:** headers and `.gitignore` meaningfully harden a static site,
> but no public web page is "unhackable." This artifact has no backend, so the
> biggest risk classes (auth, database, server code) simply don't exist yet.
> See `SECURITY_AUDIT.md` for the full assessment.
