# Why is my AI-built app slow and how do I fix it?

## Context for the AI tool

You are finding out why this app is slow and fixing the biggest causes first.

- Do not change anything until you have measured, reported your findings and I have confirmed.
- Start with check 1 and keep its numbers: every fix is judged against them.
- Report findings as a table: page or query, problem, measured cost (KB, ms or request count),
  proposed fix, expected gain. Order by expected gain.
- Fix one cause at a time and measure again after each fix. Do not refactor code that is not on
  the slow path.
- Ask before adding a dependency, changing the database schema or changing hosting settings.

## 1. Measure before you change anything

**What to look for:** the real numbers for the 2–3 pages that matter most (landing page, the main
screen after login, the slowest page users complain about). Test a production build, not the dev
server: dev mode is always slower.

```bash
npx lighthouse https://<your-app>/<page> --only-categories=performance --output=json --output-path=./lh-before.json
```

Lighthouse simulates a mid-range phone by default. Note LCP (largest contentful paint), CLS
(layout shift), TBT (total blocking time) and total JavaScript transferred. In the browser's
Network tab, also note the request count and the slowest requests on each page.

**How to fix:** nothing yet. Write the numbers down in your report.

**How to verify:** you have a before-table with LCP, CLS, TBT, JS size and request count per page.

## 2. Images

**What to look for:** the most common cause we see. Photos uploaded as 3–5 MB PNG or JPEG and
shown at 400 pixels wide, no `width`/`height` (layout jumps), every image loading at once, or the
hero image lazy-loaded (which delays LCP).

```bash
find . -path ./node_modules -prune -o -type f \( -iname "*.png" -o -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.gif" \) -size +200k -print
```

Also check images coming from storage or a CMS in the Network tab (filter: Img, sort by size).

**How to fix:**

- Resize to about twice the largest displayed width and convert photos to WebP or AVIF.
- Give every `<img>` a `width` and `height` (or a fixed aspect ratio) to stop layout shift.
- Add `loading="lazy"` and `decoding="async"` to images below the fold.
- For the main above-the-fold image: no lazy loading, add `fetchpriority="high"`.
- Use `srcset` and `sizes` so phones get small files. In Next.js, use `next/image` with a correct
  `sizes` prop and mark the hero image for early loading (`preload` in Next.js 16 and later,
  `priority` in older versions: check the installed version). For images from Supabase Storage, the image
  transformation API can resize on the fly (paid plans).

**How to verify:** Lighthouse no longer lists "Properly size images" or "Serve images in modern
formats" for these pages, and total image bytes per page dropped.

## 3. One big JavaScript bundle

**What to look for:** Lovable and Bolt apps are single-page apps: by default every screen,
library and chart ships on the first visit. Common culprits: chart and editor libraries, full
`lodash` or `moment`, whole icon packs, PDF or Excel libraries, admin screens loaded for visitors. See what is in the bundle:

- Vite apps (Lovable, Bolt) and SvelteKit: add `visualizer()` from `rollup-plugin-visualizer` to
  the `plugins` in `vite.config`, run `npm run build` and open the generated `stats.html`. For a
  plain Vite single-page app, `npx vite-bundle-visualizer` does the same without a config change.
- Next.js: builds use Turbopack by default from Next.js 16. Use the bundle analyzer built into
  your Next.js version if it has one (see "Analyzing bundles" in the Next.js docs), or build with
  `next build --webpack` and use `@next/bundle-analyzer` (`ANALYZE=true`).

**How to fix:**

- Split by route: `React.lazy(() => import('./pages/Admin'))` with `<Suspense>`, `next/dynamic`
  for heavy components, or dynamic `import()` when a feature is used (export to PDF, charts).
- Import only what you use (`import debounce from 'lodash/debounce'`, named icon imports).
- Replace `moment` with `date-fns` or the built-in `Intl` API. Remove unused dependencies.
- Keep third-party scripts (chat widgets, trackers) off the critical path: load them after the page
  is interactive.

**How to verify:** run the analyzer and Lighthouse again: JavaScript transferred on the first page
and TBT both went down.

## 4. Queries in loops and fetching everything

**What to look for:** N+1 queries (one query for a list, then one per item), `select('*')` on wide
tables, no `limit` or pagination, data filtered or counted in JavaScript after loading the whole
table, and requests that wait for each other when they don't need to.

```ts
// N+1: one request per post
for (const post of posts) {
  const { data } = await supabase.from('profiles').select('name').eq('id', post.author_id);
}
```

**How to fix:**

- Fetch related data in one query. Supabase: `.select('id, title, author:profiles(name)')`.
  SQL: a `join`. Or batch with one `in` query: `.in('id', authorIds)`.
- Select only the columns the screen shows and paginate: `.range(0, 19)` or `limit 20`.
- Count and filter in the database (`{ count: 'exact', head: true }`, `where`), not in the browser.
- Run independent requests in parallel with `Promise.all`.

**How to verify:** the Network tab shows fewer requests for the page and the slowest request is
faster than in the before-table.

## 5. Missing database indexes

**What to look for:** queries that scan the whole table. They are fast with 100 rows and slow with
100,000. Columns used in `where`, `order by`, joins and RLS policies (like `user_id`) need an index.
On Supabase or any Postgres with `pg_stat_statements`:

```sql
select calls, round(mean_exec_time::numeric, 1) as mean_ms,
       round(total_exec_time::numeric) as total_ms, left(query, 120) as query
from pg_stat_statements
order by total_exec_time desc
limit 15;
```

Foreign key columns without an index (single-column keys):

```sql
select c.conrelid::regclass as table_name, a.attname as column_name
from pg_constraint c
join pg_attribute a on a.attrelid = c.conrelid and a.attnum = c.conkey[1]
where c.contype = 'f'
  and not exists (
    select 1 from pg_index i
    where i.indrelid = c.conrelid and i.indkey[0] = c.conkey[1]
  );
```

Run `explain analyze <query>` for the slowest ones: `Seq Scan` on a large table is the sign.

**How to fix:** add an index per hot column or column pair in a migration, for example
`create index if not exists notes_user_id_idx on public.notes (user_id);`. On a large live table
use `create index concurrently` (it cannot run inside a transaction, so run it on its own).
Supabase's Performance Advisor also lists unindexed foreign keys.

**How to verify:** `explain analyze` shows an `Index Scan` and `mean_ms` for that query drops.

## 6. Caching

**What to look for:** hashed static files (`/assets/index-a1b2c3.js`) served without a long cache,
the same public data fetched again on every navigation, and public pages rendered from scratch
on every request.

```bash
curl -sI https://<your-app>/assets/<file>.js | grep -i cache-control
```

**How to fix:** hashed assets: `Cache-Control: public, max-age=31536000, immutable` (most hosts do
this, check yours). Public data that changes rarely: cache it on the server or CDN (for example
`revalidate` in Next.js, `setHeaders` with `s-maxage` in SvelteKit) or in a client query cache
such as TanStack Query with a sensible `staleTime`. Never cache responses that contain one
user's private data on a shared CDN.

**How to verify:** a second visit loads assets from cache (Network tab: "disk cache" or `304`),
and the after-table from check 1 shows the gain for each page.

---

Still stuck? WDDW fixes it. Book a free 15-minute call:
https://www.wedodevwork.com/en/contact-us?utm_source=fixes&utm_medium=md&utm_campaign=slow-ai-built-app

Did this help? An honest review helps us a lot: https://g.page/r/CXmlHMBM4uxyEBM/review
More free fixes: https://www.wedodevwork.com/en/fixes · Star the repo: https://github.com/WEDODEVWORK/fixes
