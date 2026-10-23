# How to stop your AI tool from breaking things it already fixed

## Context for the AI tool

You are setting up a project instructions file for this repository, using the template below.

- Do not write any file until you have shown me the filled-in template and I have confirmed.
- Fill in the template from what is really in this project: read `package.json` (or the
  equivalent), the folder structure, the test and lint setup and any existing instructions file.
  Never invent a command. If something is missing (for example there are no tests), say so.
- If an instructions file already exists, merge into it: keep the project facts that are there
  and report any rule that conflicts with the template.
- Keep the final file under about 100 lines. Short rules get followed; long ones get skimmed.

## 1. Pick the right file

**What to look for:** which tools the team uses and which instructions files already exist
(`CLAUDE.md`, `AGENTS.md`, `.cursor/rules/`, `.cursorrules`, `.github/copilot-instructions.md`).

**How to fix:** Claude Code reads `CLAUDE.md` in the project root. Cursor, Codex and many other
agents read `AGENTS.md`. To keep one source, put the rules in `AGENTS.md` and make `CLAUDE.md` a
single line: `@AGENTS.md` (Claude Code imports the file). Lovable has no repo file for this: paste
the rules into the project's Knowledge in its settings.

**How to verify:** exactly one file holds the rules, and every tool in use points at it.

## 2. Find the real commands

**What to look for:** the commands that prove a change works: tests, type check, lint, build.

```bash
cat package.json | grep -A20 '"scripts"'
```

**How to fix:** put the exact commands in the template. If there is no test runner, tell me and
propose a minimal setup (for example Vitest for a Vite or SvelteKit app) as a separate task. Do
not add it now.

**How to verify:** run each command once and report that it works as written. The dev server
does not exit on its own: start it, check that the app loads, then stop it.

## 3. Fill in and add the template

**What to look for:** every `<placeholder>` in the template below replaced with facts from this
project, and sections that do not apply removed.

**How to fix:** after I confirm, write the file.

```markdown
# Project rules

## Project
- Stack: <framework, language, database, hosting>
- Structure: <where pages, components, server code and migrations live>
- Conventions: <styling approach, naming, state management, anything a newcomer gets wrong>

## Commands
- Install: <command>
- Dev server: <command>
- Tests: <command>
- Type check: <command>
- Lint: <command>
- Build: <command>

## Before you say a task is done
- Run the tests, type check and lint above. Show the result. If something fails, say so: never
  report "done" or "fixed" on a failing or unrun check.
- Say how you verified the change (test, command, or the page and steps you checked).
- List every file you changed.

## Stay inside the task
- Change only what the task needs. No unrelated refactors, renames, reformatting or
  "improvements" to other code.
- Do not touch files outside the task. If another file must change, tell me why first.
- Do not delete or rewrite code you do not understand. Ask.
- Ask before adding a dependency.

## Keep fixed bugs fixed
- Every bug fix comes with a test that fails before the fix and passes after it.
- Never delete, skip or weaken a test to make it pass. If a test looks wrong, ask.
- Read the "Fixed before" list below before changing the related code. Do not undo those fixes.

## Secrets
- Never commit `.env` files, keys or passwords. New variables go in `.env.example` with a
  placeholder value.
- Secret keys never go in frontend code or in variables with a public prefix
  (<the public prefix of this framework, for example VITE_ or NEXT_PUBLIC_>).
- Never print secrets in logs, output or messages.

## Database
- Ask before any schema change, migration, policy change or data deletion.
- Schema changes go in a new migration file in <migrations folder>. Never edit an old migration.
- <If Supabase: every new table gets RLS enabled and policies in the same migration.>

## Never without asking
- Destructive commands: dropping or truncating tables, resetting the database, `rm -rf`,
  `git push --force`, `git reset --hard`.
- Changes to auth, payments, or deployment and hosting settings.

## Fixed before (do not undo)
- <YYYY-MM-DD> <what broke> -> <what fixed it, file> (test: <test name>)
```

**How to verify:** the file has no `<placeholder>` left, and every command in it was run in check 2.

## 4. Fill the "Fixed before" list

**What to look for:** bugs that came back before. Search the git history and ask me:

```bash
git log --oneline -i --grep="fix" | head -40
```

**How to fix:** add one line per bug that matters: the date, what broke, what fixed it and the
test that guards it. Where a fix has no test yet, propose one (as a separate task) so the rule
"keep fixed bugs fixed" is enforced by the test suite, not only by the file.

**How to verify:** each line in the list names a file and, where one exists, a test.

## 5. Check the tool actually follows it

**What to look for:** proof that the rules are loaded, not just present.

**How to fix:** if the tool ignores the file, check its name and location (project root), that
it is committed, and that it is short enough. Move rarely needed detail to separate docs and link
to them.

**How to verify:** start a fresh session and ask: "What must you do before you say a task is done,
and which files may you change?" The answer should quote the rules. Then give it a small real task
and check it runs the tests and lists the files it changed.

---

Still stuck? WDDW fixes it. Book a free 15-minute call:
https://www.wedodevwork.com/en/contact-us?utm_source=fixes&utm_medium=md&utm_campaign=claude-md-starter

Did this help? An honest review helps us a lot: https://g.page/r/CXmlHMBM4uxyEBM/review
More free fixes: https://www.wedodevwork.com/en/fixes · Star the repo: https://github.com/WEDODEVWORK/fixes
