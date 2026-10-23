# How to check if your Supabase data is public

## Context for the AI tool

You are checking this project's Supabase database for data that anyone on the internet can read
or change. The anon (or `sb_publishable_`) key is in the frontend bundle by design, so every
permission it has is public.

- Do not change anything until you have reported your findings and I have confirmed.
- Work through the checks in order. For each check, show the query or command you ran and its result.
- Report findings first as a table: object, problem, risk (high / medium / low), proposed fix.
- When I confirm, write fixes as a new SQL migration in the project's migrations folder (or as SQL
  I can paste into the SQL editor if there is no migrations folder). Never edit old migrations.
- Never use the service-role key in frontend code to "fix" a permission error.

Run the SQL below in the Supabase SQL editor (or with `supabase db` / `psql` against the project).
If the project exposes more schemas than `public` (Settings → API → Exposed schemas), repeat each
query for those schemas.

## 1. Tables without Row Level Security

**What to look for:** tables in an exposed schema with `rowsecurity = false`. With the default
grants, the anon role can select, insert, update and delete every row in them.

```sql
select schemaname, tablename, rowsecurity
from pg_tables
where schemaname = 'public'
order by rowsecurity, tablename;
```

Also check what the anon and authenticated roles are granted:

```sql
select table_name, grantee, string_agg(privilege_type, ', ' order by privilege_type) as privileges
from information_schema.role_table_grants
where table_schema = 'public' and grantee in ('anon', 'authenticated')
group by table_name, grantee
order by table_name, grantee;
```

**How to fix:** enable RLS on every table in an exposed schema, and add the policies the app
needs in the same migration. RLS with no policies blocks all access through the API (the
service-role key still bypasses it), so a table the app reads will look empty until a policy exists.

```sql
alter table public.notes enable row level security;

create policy "Users read their own notes" on public.notes
  for select to authenticated using ((select auth.uid()) = user_id);
create policy "Users add their own notes" on public.notes
  for insert to authenticated with check ((select auth.uid()) = user_id);
create policy "Users edit their own notes" on public.notes
  for update to authenticated
  using ((select auth.uid()) = user_id) with check ((select auth.uid()) = user_id);
create policy "Users delete their own notes" on public.notes
  for delete to authenticated using ((select auth.uid()) = user_id);
```

Tables that only the server uses (logs, payments, internal settings) need RLS enabled and no
policies at all. If anon must never touch a table, also run `revoke all on public.<table> from anon;`.

**How to verify:** the first query returns `rowsecurity = true` for every table. The Security
Advisor in the dashboard (Advisors → Security) no longer shows "RLS disabled in public".

## 2. Policies that let everyone in

**What to look for:** RLS is on, but a policy makes it meaningless:

- `qual` or `with_check` is `true` for the `anon` or `public` role (`public` includes anon).
- `using (true)` for `authenticated` on private data. Sign-up is usually open, so "any logged-in
  user" means "anyone who signs up".
- Policies that read `auth.jwt() -> 'user_metadata'`. Users can edit their own user metadata, so
  it must never decide access. Use `app_metadata` or a table you control.
- An `insert` policy with `with check (true)`, or an `update` policy whose `with check` is looser
  than its `using`. Both let a user write rows they do not own or move a row to another
  `user_id`.

```sql
select tablename, policyname, roles, cmd, qual, with_check
from pg_policies
where schemaname = 'public'
order by tablename, cmd;
```

**How to fix:** replace each broad policy with one tied to the owner, team or role, as in check 1.
Keep `using (true)` only for data that is meant to be public (published posts, a product
catalogue), and only for `select`. If a table mixes public and private columns (a `profiles` table
with names and phone numbers), move the private columns to a separate table with its own policies.

**How to verify:** read the policy list again with me and confirm each remaining `true` is intended
public data. Then run the anon test in check 5.

## 3. Views and functions that skip RLS

**What to look for:** views run with the permissions of their owner (usually `postgres`), so a
view over a protected table can expose it. `security definer` functions run as their owner too and
anyone can call them through `/rest/v1/rpc/<name>` unless execute is revoked.

```sql
select c.relname as view_name, c.reloptions
from pg_class c join pg_namespace n on n.oid = c.relnamespace
where n.nspname = 'public' and c.relkind in ('v', 'm');

-- security definer functions, skipping ones that belong to extensions
select p.proname, pg_get_function_identity_arguments(p.oid) as args,
       has_function_privilege('anon', p.oid, 'execute') as anon_can_execute
from pg_proc p join pg_namespace n on n.oid = p.pronamespace
where n.nspname = 'public' and p.prosecdef
  and not exists (select 1 from pg_depend d where d.objid = p.oid and d.deptype = 'e')
order by p.proname;
```

**How to fix:** on Postgres 15 or later, make views respect RLS with
`alter view public.<view> set (security_invoker = true);`. Materialized views cannot use RLS:
move them out of exposed schemas or revoke select from anon and authenticated. For a security
definer function that anon should not call: `revoke execute on function public.<name>(<args>) from public, anon;`.
Functions that must stay security definer should check `auth.uid()` inside and set
`set search_path = ''` (or a fixed schema).

**How to verify:** views show `security_invoker=true` in `reloptions`; the function query shows
`anon_can_execute = false` for everything that is not meant to be public.

## 4. Storage buckets

**What to look for:** public buckets holding private files (invoices, IDs, user uploads), and
policies on `storage.objects` that allow listing or uploading for anyone.

```sql
select id, name, public from storage.buckets;

select policyname, roles, cmd, qual, with_check
from pg_policies where schemaname = 'storage' and tablename = 'objects';
```

**How to fix:** make private buckets non-public and serve files through signed URLs
(`createSignedUrl`). Scope policies to the user's folder, for example
`(storage.foldername(name))[1] = (select auth.uid())::text`.

**How to verify:** a file URL under `/storage/v1/object/public/<bucket>/...` for a private bucket
returns an error instead of the file.

## 5. Test it as a stranger

**What to look for:** proof from the outside. Take the project URL and anon key from the frontend
(or `.env`) and request each table without logging in:

```bash
curl "https://<project-ref>.supabase.co/rest/v1/<table>?select=*&limit=3" \
  -H "apikey: <anon-key>" -H "Authorization: Bearer <anon-key>"
```

With an `sb_publishable_` key, send only the `apikey` header. Rows in the response mean the table
is public. `[]` means no rows are visible (check the table is not simply empty). A `401` or
`permission denied` error means anon has no grant.

The write test below really inserts a row if the table is open. Ask me before running it, run it
against a staging project or a Supabase branch when one exists, and if it inserts a row, report
the table and the new row's id so it can be deleted:

```bash
curl -X POST "https://<project-ref>.supabase.co/rest/v1/<table>" \
  -H "apikey: <anon-key>" -H "Authorization: Bearer <anon-key>" \
  -H "Content-Type: application/json" -H "Prefer: return=representation" -d '{}'
```

**How to fix:** go back to check 1 or 2 for any table that answers with data or accepts the write.

**How to verify:** repeat the test for every table and list the results. Then log in as a normal
test user and confirm the app still works: their own data loads, other users' data does not.

---

Still stuck? WDDW fixes it. Book a free 15-minute call:
https://www.wedodevwork.com/en/contact-us?utm_source=fixes&utm_medium=md&utm_campaign=supabase-rls-check

Did this help? An honest review helps us a lot: https://g.page/r/CXmlHMBM4uxyEBM/review
More free fixes: https://www.wedodevwork.com/en/fixes · Star the repo: https://github.com/WEDODEVWORK/fixes
