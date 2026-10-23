# Free fixes for AI-built apps

Markdown files you paste into Cursor, Claude Code or Lovable. Your AI tool runs the checks,
reports what it finds and fixes it. Written and tested by the developers at
[We Do Dev Work](https://www.wedodevwork.com/en/fixes) in Bangkok.

## How to use

1. Open the fix you need and download or copy the `.md` file.
2. Paste it into your AI tool (or add it to your project) and ask it to follow the file.
3. Read its findings before you let it change anything.

## Fixes

- [How to check if your Supabase data is public](supabase-rls-check/supabase-rls-check.md): Find the tables, views, functions and buckets anyone with your anon key can read, and lock them down with RLS. ([on the site](https://www.wedodevwork.com/en/fixes/supabase-rls-check))
- [How to do a security check before you launch your AI-built app](before-launch-security-pass/before-launch-security-pass.md): Find secret keys in the frontend, committed .env files, open API routes, missing rate limits and loose CORS before real users arrive. ([on the site](https://www.wedodevwork.com/en/fixes/before-launch-security-pass))
- [Why is my AI-built app slow and how do I fix it?](slow-ai-built-app/slow-ai-built-app.md): Measure first, then fix the usual causes in AI-built apps - huge images, one big JavaScript bundle, queries in loops, missing indexes and no caching. ([on the site](https://www.wedodevwork.com/en/fixes/slow-ai-built-app))
- [How to set up Stripe webhooks that actually work](stripe-webhooks-that-work/stripe-webhooks-that-work.md): Verify signatures on the raw body, handle retries and duplicates, answer fast, and keep test and live keys apart so payments and subscriptions stay in sync. ([on the site](https://www.wedodevwork.com/en/fixes/stripe-webhooks-that-work))
- [How to stop your AI tool from breaking things it already fixed](claude-md-starter/claude-md-starter.md): A ready-to-copy CLAUDE.md / AGENTS.md with rules that stop regressions - run the tests before saying done, stay inside the task, keep fixed bugs fixed, never commit secrets. ([on the site](https://www.wedodevwork.com/en/fixes/claude-md-starter))

More fixes and updates: https://www.wedodevwork.com/en/fixes. If a fix helped, a star helps others find it.
