# How to do a security check before you launch your AI-built app

## Context for the AI tool

You are doing a security pass on this project before it goes live. The goal is to find secrets
that leak, endpoints that answer anyone, and abuse that costs money.

- Do not change anything until you have reported your findings and I have confirmed.
- Go through the checks in order. For each one, show the command you ran and what it returned.
- Report findings as a table: where (file and line or URL), problem, risk (high / medium / low),
  proposed fix. Put leaked secrets first.
- Never print the full value of a secret you find. Show the first 6 characters and the file.
- Ask me before rotating keys, rewriting git history or changing auth settings.

## 1. Secret keys in the frontend

**What to look for:** server-only keys that end up in code the browser downloads. Anything in an
environment variable with a public prefix (`NEXT_PUBLIC_`, `VITE_`, `PUBLIC_`, `EXPO_PUBLIC_`,
`REACT_APP_`) is shipped to every visitor. Dangerous ones: Supabase service-role or `sb_secret_`
keys, Stripe `sk_live_`/`sk_test_`/`rk_` keys, OpenAI/Anthropic keys, database URLs, SMTP or
Resend keys, AWS keys.

```bash
grep -rnE "(NEXT_PUBLIC|VITE|PUBLIC|EXPO_PUBLIC|REACT_APP)_[A-Z_]*(SECRET|SERVICE|PRIVATE|PASSWORD|TOKEN|KEY)" \
  --include='*.ts' --include='*.tsx' --include='*.js' --include='*.jsx' --include='*.svelte' \
  --include='*.vue' --include='.env*' \
  --exclude-dir=node_modules --exclude-dir=.next --exclude-dir=.svelte-kit --exclude-dir=dist \
  --exclude-dir=build . | cut -c1-120
```

Public anon or publishable keys (`..._ANON_KEY`, `..._PUBLISHABLE_KEY`, Stripe `pk_`) are meant to
be public. Report everything else.

Then build the app and search the output the browser receives (`dist`, `build`, `.next/static`,
`.svelte-kit/output/client` or `.output/public`, depending on the framework):

```bash
grep -rnoE "sk_(live|test)_[A-Za-z0-9]{10,}|rk_live_[A-Za-z0-9]{10,}|sb_secret_[A-Za-z0-9_-]{10,}|sk-[A-Za-z0-9_-]{20,}|AKIA[0-9A-Z]{16}|postgres(ql)?://[^\"' ]+" <build-output-folder>
grep -rnoE "eyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+" <build-output-folder> | sort -u
```

For each JWT found, base64-decode the middle part and check the `role` claim. `anon` is expected.
`service_role` is a full database bypass.

**How to fix:** move the key to a server-only variable (no public prefix) and call the third-party
API from a server route, server action or edge function. Then rotate the key in the provider's
dashboard: once it was in a deployed bundle, assume it is copied.

**How to verify:** rebuild and run both searches again: no secrets in the output. The old key is
revoked in the provider's dashboard.

## 2. .env files in git

**What to look for:** `.env`, `.env.local` or `.env.production` committed now or at any point in
history (`.env.example` with placeholder values is fine).

```bash
git ls-files | grep -E "(^|/)\.env" | grep -v "\.example$"
git log --all --oneline --name-only -- '.env*' ':(glob)**/.env*' | head -50
grep -nE "^\.env" .gitignore
```

**How to fix:** add `.env*` (and `!.env.example`) to `.gitignore`, run `git rm --cached <file>`,
and rotate every key that was in the file. Removing it from history (`git filter-repo`) is optional
and needs my approval; rotation is not optional, especially if the repo was ever public or shared.

**How to verify:** `git ls-files` shows no `.env` files; every key from the old file has been
rotated.

## 3. API routes that answer anyone

**What to look for:** list every server entry point: `app/**/route.ts`, `pages/api/**`, Next.js
server actions (`"use server"`, these are public POST endpoints), SvelteKit `+server.ts` and form
actions, Express/Hono routes, Supabase or Firebase functions. For each, answer two questions:

1. Does it check who is calling (a valid session or token) before doing anything?
2. Does it check that this caller may touch this record? Taking a `userId`, `orgId` or `orderId`
   from the request and using it without checking ownership lets anyone read other users' data by
   changing the number.

Also check for trusted client input: prices, discounts, roles or `isAdmin` read from the request
body instead of the server.

**How to fix:** add an auth check at the top of each private handler that returns `401` without a
valid session. Take the user id from the session, never from the request. Check ownership in the
query (`where id = $1 and user_id = $session_user`). Read prices and roles from the database.
Supabase edge functions deployed with `--no-verify-jwt` must verify the caller themselves.

**How to verify:** test against a local, preview or staging deployment, not production. Call
each private endpoint without a cookie or token and expect `401` or `403`. These calls can still
run the handler if it is open, so ask me before calling routes that send email or SMS, take
payments, delete data or call a paid API:

```bash
curl -i -X POST https://<your-app>/api/<route> -H "Content-Type: application/json" -d '{}'
```

Log in as test user A and request user B's record by id: expect `403` or `404`.

## 4. No rate limits on expensive or sensitive actions

**What to look for:** login, sign-up, password reset, magic links, contact forms, anything that
sends email or SMS, file uploads and every route that calls a paid AI API. Without limits, one
script can run up your AI bill, burn your email reputation or brute-force passwords.

**How to fix:** add a limit per IP and per user on those routes (for example a Redis- or
database-backed limiter, or the rate-limiting rules of your host or WAF). Set a monthly spending
cap in your AI provider's dashboard. On Supabase, review the Auth rate limits (Authentication →
Rate Limits) and enable CAPTCHA on sign-up if bots are a risk.

**How to verify:** ask me first, and run this only against a local, preview or staging
deployment. Pick a harmless route behind the same limiter (for example login with a wrong
password). Never point it at a route that sends email or SMS or calls a paid API unless I confirm
and the provider is in test mode. Send 30 quick requests and count the status codes. You should
see `429`s:

```bash
for i in $(seq 1 30); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST https://<your-app>/api/<route> \
    -H "Content-Type: application/json" -d '{}'
done | sort | uniq -c
```

## 5. CORS that trusts every site

**What to look for:** `Access-Control-Allow-Origin: *` on endpoints that return private data, or
code that copies the request's `Origin` header into the response together with
`Access-Control-Allow-Credentials: true`. That second pattern lets any website make logged-in
requests on behalf of your users.

```bash
curl -s -D - -o /dev/null -H "Origin: https://evil.example" https://<your-app>/api/<route>
```

**How to fix:** allow a fixed list of your own origins. Only send `Allow-Credentials: true` for
origins on that list. Public, read-only endpoints may use `*` without credentials.

**How to verify:** the curl above does not return `Access-Control-Allow-Origin: https://evil.example`.

## 6. Errors and packages that leak

**What to look for:** API responses that return stack traces, SQL errors or full error objects
to the browser; debug routes or admin pages left on; known vulnerable packages.

```bash
grep -rnE "\.stack\b|JSON\.stringify\((err|error|e)\)" \
  --include='*.ts' --include='*.tsx' --include='*.js' --include='*.jsx' --include='*.svelte' \
  --exclude-dir=node_modules . | head -40
npm audit --omit=dev
```

**How to fix:** log the full error on the server and return a short generic message with a
status code. Remove or protect debug and admin routes. Update packages with high or critical
advisories (`npm audit fix`, then retest).

**How to verify:** trigger an error (bad input to a form) and check the browser's Network tab: no
stack trace or SQL. `npm audit --omit=dev` shows no high or critical issues.

## 7. Database rules

**What to look for:** if the app talks to Supabase or Firebase from the browser, the database
rules are your real backend. Check that every table or collection has rules and that they are tied
to the logged-in user, not `true` or `allow read, write: if true`.

**How to fix:** for Supabase, enable RLS and write owner-based policies. For Firebase, write
rules that compare `request.auth.uid` to the document's owner.

**How to verify:** query the data with only the public key and no login: private data must not
come back.

---

Still stuck? WDDW fixes it. Book a free 15-minute call:
https://www.wedodevwork.com/en/contact-us?utm_source=fixes&utm_medium=md&utm_campaign=before-launch-security-pass

Did this help? An honest review helps us a lot: https://g.page/r/CXmlHMBM4uxyEBM/review
More free fixes: https://www.wedodevwork.com/en/fixes · Star the repo: https://github.com/WEDODEVWORK/fixes
