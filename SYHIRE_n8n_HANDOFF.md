# SyHire × n8n — Integration & Security Handoff

**Give this whole file to Claude Code.** It describes how to connect the existing
SyHire site to n8n *securely*, plus a security checklist. Written for a Next.js
app deployed on Vercel with n8n as the hidden automation engine.

---

## 0. Context

- The current SyHire site is a **static self-contained page** (`SyHire.standalone.html`),
  design only — no backend.
- We now have a running **n8n** instance and want the site to trigger real
  automations (publish job, notify recruiter, AI screening, etc.).
- **Hard rule:** the browser must NEVER hold the n8n API key. All privileged
  calls go through our own backend (a thin proxy). The browser may only call
  **n8n Webhook URLs** (public-by-design trigger endpoints) or, better, our own
  API which then calls n8n.

---

## 1. Target Architecture

```
Browser (Next.js frontend)
   │  fetch('/api/automations/run', {action, payload})   ← same-origin only
   ▼
Next.js API Route  (server-side, holds secrets)
   │  - verify Supabase session (user is logged in)
   │  - rate-limit + validate input
   │  - map business action → n8n workflow/webhook
   ▼
n8n REST API / Production Webhook   (server-to-server, with N8N_API_KEY)
   ▼
n8n Execution Engine
```

**Why:** the API key and n8n URL stay on the server. The browser only ever talks
to our own `/api/*` on the same domain. No secret is ever shippable to a user.

---

## 2. Environment Variables (Vercel → Project → Settings → Environment Variables)

Never hardcode these. Never expose them with the `NEXT_PUBLIC_` prefix.

```
N8N_BASE_URL          = https://<your>.app.n8n.cloud     # or self-hosted URL
N8N_API_KEY           = <n8n API key>                    # server-only secret
N8N_WEBHOOK_SECRET    = <random 32+ char string>         # HMAC to sign requests
SUPABASE_URL          = https://<project>.supabase.co
SUPABASE_ANON_KEY     = <anon key>                       # ok on client
SUPABASE_SERVICE_ROLE = <service role>                   # SERVER ONLY, never client
```

Generate secrets: `openssl rand -hex 32`.

---

## 3. Backend Proxy — `/app/api/automations/run/route.ts`

```ts
import { NextRequest, NextResponse } from 'next/server';
import { createServerClient } from '@supabase/ssr';
import crypto from 'crypto';

// Business action → n8n webhook path. Extend as you add workflows.
const ACTION_MAP: Record<string, string> = {
  'publish_job':      '/webhook/publish-job',
  'notify_recruiter': '/webhook/notify-recruiter',
  'ai_screen_resume': '/webhook/ai-screen',
  'schedule_interview':'/webhook/schedule-interview',
};

export async function POST(req: NextRequest) {
  // 1. Require an authenticated Supabase user
  const supabase = createServerClient(/* ...cookies wiring... */);
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  // 2. Validate input
  const { action, payload } = await req.json().catch(() => ({}));
  const path = ACTION_MAP[action];
  if (!path) return NextResponse.json({ error: 'Unknown action' }, { status: 400 });

  // 3. Sign the request so n8n can verify it really came from us
  const body = JSON.stringify({ action, payload, userId: user.id, ts: Date.now() });
  const sig = crypto
    .createHmac('sha256', process.env.N8N_WEBHOOK_SECRET!)
    .update(body)
    .digest('hex');

  // 4. Call n8n server-to-server (key/secret never leave the server)
  const res = await fetch(`${process.env.N8N_BASE_URL}${path}`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Signature': sig,
      'X-Api-Key': process.env.N8N_API_KEY!,
    },
    body,
  });

  if (!res.ok) return NextResponse.json({ error: 'Automation failed' }, { status: 502 });
  return NextResponse.json({ ok: true, result: await res.json().catch(() => null) });
}
```

### In n8n
- Each workflow starts with a **Webhook** node (method POST, the path above).
- **First node after the webhook:** a **Code/IF node that verifies `X-Signature`**
  by recomputing the HMAC with the same secret. Reject if it doesn't match.
  This stops randoms from hitting your webhook URL directly.
- Return a small JSON so the UI can show success/failure.

---

## 4. Frontend call (same-origin, no secrets)

```ts
async function runAutomation(action: string, payload: unknown) {
  const res = await fetch('/api/automations/run', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ action, payload }),
  });
  if (!res.ok) throw new Error('Automation failed');
  return res.json();
}
// e.g. on "Publish Job" button:
await runAutomation('publish_job', { jobId, channels: ['website','facebook','linkedin'] });
```

---

## 5. Migrate the current design into the app

- Keep the existing SyHire visual design; port `SyHire.standalone.html`'s markup/styles
  into Next.js components (or embed as a first pass).
- Replace the fake front-end login with **Supabase Auth** (email magic-link or
  password). Gate the dashboard with `middleware.ts` that checks the session.
- Store jobs, candidates, workflows in **Supabase Postgres** with **Row Level
  Security (RLS)** ON for every table.

---

## 6. SECURITY CHECKLIST (do all of these)

**Secrets**
- [ ] n8n API key & Supabase service role are **server-only** env vars. Never
      `NEXT_PUBLIC_`, never in client bundles, never in the repo.
- [ ] `.env*` is gitignored (already is).

**Auth & access**
- [ ] Supabase Auth for real login; remove the demo/no-password gate.
- [ ] Enable **RLS** on all tables; write policies so users see only their org's data.
- [ ] Role-based permissions (admin / recruiter / viewer) enforced **server-side**,
      not just hidden in the UI.

**n8n**
- [ ] Every webhook verifies the **HMAC signature** — reject unsigned/mismatched.
- [ ] Use n8n **production** webhook URLs (obscure, unguessable paths).
- [ ] Put n8n behind auth if self-hosted; don't expose the n8n editor publicly.
- [ ] Never call n8n directly from the browser with the API key.

**Transport & headers**
- [ ] Keep the hardened `vercel.json` headers (CSP, HSTS, frame-deny already set).
- [ ] `connect-src` in CSP must include ONLY your own origin — the browser talks to
      `/api/*`, not to n8n directly. (Tighten it back to `'self'` once the proxy
      is in; the n8n domains I added are only needed if you call webhooks from the
      browser, which the proxy replaces.)

**Abuse protection**
- [ ] **Rate-limit** `/api/automations/*` (e.g. Upstash Ratelimit or Vercel KV) —
      per user and per IP.
- [ ] **Validate & size-limit** every input (use `zod`). Reject unexpected fields.
- [ ] Log automation runs (who/what/when) to Supabase for auditing.
- [ ] Add a global error boundary; never leak stack traces or secrets in responses.

**Deploy**
- [ ] Keep the GitHub repo **private**.
- [ ] Enable Vercel **Deployment Protection** if the app shouldn't be public yet.
- [ ] Rotate any secret that was ever pasted in chat, a screenshot, or committed.

---

## 7. Build order (suggested)

1. Next.js app + Supabase Auth + one protected dashboard page.
2. Port SyHire design in.
3. `/api/automations/run` proxy + ONE working action (`publish_job`) end-to-end.
4. HMAC verification in the n8n webhook.
5. Rate limiting + input validation + audit logging.
6. Add remaining actions/engines; then the AI workflow builder.

> Ship #1–#4 first and confirm one automation works end-to-end before expanding.
