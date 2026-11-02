# 🔐 Environment Variables Cheatsheet

> A practical reference for getting every API key, secret and connection string a modern full-stack (Next.js) app needs: where to find it, how to generate it, and how to avoid the usual mistakes.

**By Biyinzika Bruce**

![License](https://img.shields.io/badge/license-MIT-green) ![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen)

---

## 📚 Table of Contents

- [Ground Rules](#-ground-rules)
- **Core**
  - [1. App URL](#1-app-url)
  - [2. Database: Neon PostgreSQL](#2-database--neon-postgresql)
  - [3. Better Auth](#3-better-auth)
- **OAuth & APIs**
  - [4. Google OAuth](#4-google-oauth)
  - [5. GitHub OAuth](#5-github-oauth)
  - [6. GitHub Personal Access Token](#6-github-personal-access-token)
- **Email**
  - [7. Resend](#7-resend)
- **Payments**
  - [8. Stripe](#8-stripe)
  - [9. DGateway (Mobile Money + Cards)](#9-dgateway)
- **File Storage**
  - [10. AWS S3](#10-aws-s3)
  - [11. Cloudflare R2](#11-cloudflare-r2)
  - [12. UploadThing](#12-uploadthing)
- **Cache & Background Jobs**
  - [13. Upstash Redis](#13-upstash-redis)
  - [14. Inngest](#14-inngest)
- **AI**
  - [15. Anthropic (Claude)](#15-anthropic-claude)
  - [16. OpenAI](#16-openai)
  - [17. Google Gemini](#17-google-gemini)
- [Full `.env.local` Template](#-full-envlocal-template)
- [Production Checklist](#-production-checklist)
- [Troubleshooting](#-troubleshooting)

---

## 🧭 Ground Rules

| Rule | Why it matters |
|------|----------------|
| `NEXT_PUBLIC_*` variables are **sent to the browser** | Never put a secret behind this prefix. Anyone can read it in DevTools. |
| Everything else is **server-only** | Safe for secrets, but only accessible in server code (API routes, server actions, RSC). |
| **Restart the dev server** after editing `.env.local` | Next.js reads env files only at startup. |
| Commit `.env.example`, **never** `.env.local` | Teammates get the variable names, not your secrets. |
| One set of keys **per environment** | Dev, preview and production should never share secrets. |

Quick start:

```bash
cp .env.example .env.local   # then fill in the values
```

---

## Core

### 1. App URL

```env
APP_URL=http://localhost:3000
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

- **Dev:** `http://localhost:3000`
- **Production:** your real domain, e.g. `https://myapp.com`, with **no trailing slash**.
- **Vercel previews:** Vercel injects `VERCEL_URL` automatically (without the protocol), so you can derive the URL for preview deployments:

```ts
const appUrl =
  process.env.APP_URL ??
  (process.env.VERCEL_URL ? `https://${process.env.VERCEL_URL}` : "http://localhost:3000");
```

---

### 2. Database: Neon PostgreSQL

```env
DATABASE_URL=postgresql://USER:PASSWORD@HOST/DBNAME?sslmode=require
```

**Get it:**
1. Sign in at [neon.tech](https://neon.tech) and create a project.
2. Open **Connection Details** on the dashboard.
3. Copy the connection string.

> 💡 For serverless deployments (Vercel, Netlify), pick the **pooled** connection string (the host contains `-pooler`) so you don't exhaust connections.

**Prefer local Postgres?** Use Docker:

```bash
docker run -d --name local-postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 postgres:16-alpine
```

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/myapp
```

---

### 3. Better Auth

```env
BETTER_AUTH_SECRET=
BETTER_AUTH_URL=http://localhost:3000
```

**Generate the secret:**

```bash
# macOS / Linux / Git Bash
openssl rand -base64 32

# Windows (or anywhere with Node)
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

- `BETTER_AUTH_URL` is `http://localhost:3000` in dev and your production domain in prod.
- Generate a **different secret for every environment**. Rotating it signs everyone out.

---

## OAuth & APIs

### 4. Google OAuth

```env
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
```

**Get it:**
1. Go to [Google Cloud Console](https://console.cloud.google.com) and create or select a project.
2. **APIs & Services → OAuth consent screen**: fill in the app name, support email and scopes (`email`, `profile`, `openid`).
3. **APIs & Services → Credentials → Create Credentials → OAuth client ID** → *Web application*.
4. Add:

| Field | Development | Production |
|-------|-------------|------------|
| Authorized JavaScript origins | `http://localhost:3000` | `https://myapp.com` |
| Authorized redirect URIs | `http://localhost:3000/api/auth/callback/google` | `https://myapp.com/api/auth/callback/google` |

> ⚠️ `redirect_uri_mismatch` almost always means you forgot to add the **production** redirect URI, or there's a trailing slash or `http`/`https` mismatch.

---

### 5. GitHub OAuth

```env
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
```

**Get it:**
1. GitHub → **Settings → Developer settings → OAuth Apps → New OAuth App**.
2. Homepage URL: `http://localhost:3000`
3. Authorization callback URL: `http://localhost:3000/api/auth/callback/github`
4. Click **Register**, copy the **Client ID**, then **Generate a new client secret**.

> ⚠️ The client secret is shown **only once**. Copy it immediately.
> 💡 GitHub OAuth Apps allow only one callback URL, so create a **separate OAuth App for production**.

---

### 6. GitHub Personal Access Token

Used for server-to-server calls to the GitHub API (fetching repos, stars, contributors...).

```env
GITHUB_API_TOKEN=
```

**Get it:** GitHub → **Settings → Developer settings → Personal access tokens**.

| You need... | Scope |
|-------------|-------|
| Public repo data only (stars, forks) | no scope (just a higher rate limit) |
| Public repo read/write | `public_repo` |
| Private repos | `repo` |

> 🔒 Prefer a **fine-grained token** limited to the specific repositories you need, with an expiry date.

---

## Email

### 7. Resend

```env
RESEND_API_KEY=re_xxxxxxxxxxxx
```

**Get it:**
1. Sign up at [resend.com](https://resend.com).
2. **API Keys → Create API Key** with *Sending access*.
3. **Domains → Add Domain** and add the DNS records, so you can send from your own domain in production.

> 📦 Free tier: **100 emails/day, 3,000/month**.

---

## Payments

### 8. Stripe

```env
STRIPE_SECRET_KEY=sk_test_xxxx
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_xxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxx
```

**API keys:** [Dashboard](https://dashboard.stripe.com) → **Developers → API keys** (with *Test mode* on).

**Webhooks in development** (Stripe CLI):

```bash
# install (macOS); see stripe.com/docs/stripe-cli for other OSes
brew install stripe/stripe-cli/stripe

stripe login
stripe listen --forward-to localhost:3000/api/webhooks/stripe
# → prints a whsec_... value: use it as STRIPE_WEBHOOK_SECRET
```

**Webhooks in production:**
1. **Developers → Webhooks → Add endpoint**: `https://myapp.com/api/webhooks/stripe`
2. Select events, typically:
   - `checkout.session.completed`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
3. Copy the **Signing secret** into your production env.

| Mode | Secret key | Publishable key |
|------|------------|-----------------|
| Test | `sk_test_...` | `pk_test_...` |
| Live | `sk_live_...` | `pk_live_...` |

> ⚠️ Test and live keys, products and webhooks are completely separate. Don't mix them.

---

### 9. DGateway

One API for **MTN Mobile Money, Airtel Money and Stripe** card payments.

```env
DGATEWAY_API_KEY=dgw_test_xxxx
DGATEWAY_WEBHOOK_SECRET=
DGATEWAY_BASE_URL=
```

- Test keys start with `dgw_test_`, live keys with `dgw_live_`.
- Register a webhook endpoint (e.g. `https://myapp.com/api/webhooks/dgateway`) and subscribe to:
  - `payment.succeeded`
  - `payment.failed`
  - `payout.completed`
- Always verify the webhook signature with `DGATEWAY_WEBHOOK_SECRET` before trusting the payload.

---

## File Storage

> Pick **one** provider. You don't need all three.

### 10. AWS S3

```env
AWS_S3_REGION=us-east-1
AWS_S3_BUCKET_NAME=
AWS_S3_ACCESS_KEY_ID=
AWS_S3_SECRET_ACCESS_KEY=
```

**Get it:**
1. **S3 → Create bucket**: choose a name and region, and decide whether to block public access.
2. **IAM → Users → Create user**, then attach an S3 policy (ideally scoped to *this bucket only*).
3. **Security credentials → Create access key** → *Application running outside AWS*.
4. Bucket → **Permissions → CORS**:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST", "DELETE"],
    "AllowedOrigins": ["http://localhost:3000", "https://myapp.com"],
    "ExposeHeaders": ["ETag"]
  }
]
```

---

### 11. Cloudflare R2

S3-compatible storage with **zero egress fees**.

```env
CLOUDFLARE_R2_ACCESS_KEY_ID=
CLOUDFLARE_R2_SECRET_ACCESS_KEY=
CLOUDFLARE_R2_ENDPOINT=https://<ACCOUNT_ID>.r2.cloudflarestorage.com
CLOUDFLARE_R2_BUCKET_NAME=
CLOUDFLARE_R2_PUBLIC_DEV_URL=https://pub-xxxx.r2.dev
```

**Get it:**
1. Cloudflare dashboard → **R2 → Create bucket**.
2. **R2 → Manage API tokens → Create API token**: *Object Read & Write*, scoped to your bucket.
3. Copy the Access Key ID, Secret Access Key and the endpoint (it uses your account ID).
4. Bucket → **Settings → Public access**: enable the `r2.dev` URL for development.

> 📦 Free tier: **10 GB storage + 1M Class A operations/month**.
> 💡 In production, connect a **custom domain** instead of the rate-limited `r2.dev` URL.

---

### 12. UploadThing

```env
UPLOADTHING_TOKEN=
```

- Dashboard at [uploadthing.com](https://uploadthing.com) → your app → **API Keys** → copy the token.
- **v7+** uses this single `UPLOADTHING_TOKEN`. Older setups used `UPLOADTHING_SECRET` + `UPLOADTHING_APP_ID`, which still work for legacy versions.

---

## Cache & Background Jobs

### 13. Upstash Redis

```env
UPSTASH_REDIS_REST_URL=https://xxxx.upstash.io
UPSTASH_REDIS_REST_TOKEN=
```

- [console.upstash.com](https://console.upstash.com) → **Create database** → copy the **REST URL** and **REST token**.
- **Eviction:** turn it **on** for pure caches, **off** for queues and rate-limit counters (you don't want those silently deleted).

> 📦 Free tier: **10,000 commands/day, 256 MB**.

---

### 14. Inngest

```env
INNGEST_EVENT_KEY=
INNGEST_SIGNING_KEY=
```

- [app.inngest.com](https://app.inngest.com) → **Manage → Event Keys → Create** for `INNGEST_EVENT_KEY`.
- **Manage → Signing Key** for `INNGEST_SIGNING_KEY`.
- **Local dev needs no keys**. Run the dev server:

```bash
npx inngest-cli@latest dev
```

---

## AI

### 15. Anthropic (Claude)

```env
ANTHROPIC_API_KEY=sk-ant-xxxx
```

- [console.anthropic.com](https://console.anthropic.com) → **API Keys → Create Key**.
- A payment method is required, even to use free credits.
- 💡 Set a **monthly spend limit** under billing.

### 16. OpenAI

```env
OPENAI_API_KEY=sk-xxxx
```

- [platform.openai.com](https://platform.openai.com) → add billing → **API keys → Create new secret key**.
- ⚠️ The key is shown once. Copy it immediately.

### 17. Google Gemini

```env
GEMINI_API_KEY=
```

- [aistudio.google.com](https://aistudio.google.com) → **Get API key → Create API key**.
- Gemini has a generous free tier with per-minute rate limits on the Flash models. Check the current limits in AI Studio.

---

## 📄 Full `.env.local` Template

The complete template lives in [`.env.example`](./.env.example). Copy it to get started:

```bash
cp .env.example .env.local
```

---

## ✅ Production Checklist

- [ ] `.env*` is in `.gitignore` (but `.env.example` is committed)
- [ ] Production uses **different keys** from dev (Stripe/DGateway in **live** mode)
- [ ] OAuth apps have the **production** callback URLs
- [ ] Webhook endpoints registered with **production** signing secrets
- [ ] **Spend limits** set (Anthropic, OpenAI, AWS)
- [ ] Secrets stored in your host's secret manager (Vercel, Railway, Fly...), not the repo
- [ ] Nothing secret uses the `NEXT_PUBLIC_` prefix
- [ ] Keys rotated when a teammate leaves

**Accidentally committed a secret?** Rotate it **immediately**. Deleting the commit isn't enough, because it stays in git history and forks.

---

## 🛠 Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| "Invalid API key" | Test key used in live mode (or vice versa) |
| `redirect_uri_mismatch` | Redirect URI doesn't **exactly** match what's registered |
| Variable is `undefined` | Dev server not restarted after editing `.env.local` |
| Undefined in the browser only | Missing `NEXT_PUBLIC_` prefix (or you shouldn't expose it at all) |
| Webhook signature fails | Using the CLI `whsec_` in prod, or parsing the body before verifying |
| Still undefined | Typo in the variable name. Compare with `.env.example` |

---

## 🤝 Contributing

Found an outdated step or want to add a service? PRs welcome:

1. Fork the repo
2. Add your section following the existing format (env block → how to get it → tips)
3. Add the variables to `.env.example`
4. Open a pull request

## 📝 License

MIT © 2026 **Biyinzika Bruce**. Use it, share it, improve it.

---

<sub>Inspired by JB's <a href="https://jb.desishub.com/blog/environment-variables-cheatsheet">Developer's Environment Variables Cheatsheet</a>.</sub>
