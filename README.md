# True Adam Podcast: prep site

A private, shared newsroom for Freddy, Stew and Mike for the week between episodes. It has
topic tabs, saved articles with summaries and talking points, a "this week" pile, episode
planning, and comments. The producer fills it from the database side. The hosts browse,
add their own finds, comment, and mark items used.

```
supabase/migrations/   schema, RLS, seed topics: the contract the producer writes against
supabase/tests/        schema + RLS tests (run on plain Postgres via scripts/test-db.sh)
docs/PRODUCER.md       how to insert items, plus the exact table/column list
web/                   the site: Vite + React + supabase-js, static, mobile-first
.github/workflows/     CI (DB tests + build) and GitHub Pages deploy
```

## How it fits together

- **Database**: a dedicated Supabase project (`true-adam-podcast`), separate from GovContractHunter.
  RLS is on for every table. The anon role gets nothing. Signed-in users get access only
  if their email is in `allowed_emails`. Hosts can't rewrite text they didn't add.
- **Producer**: writes with SQL through the Supabase connector or the SQL editor. That
  bypasses RLS, so no website login is needed. See [docs/PRODUCER.md](docs/PRODUCER.md).
- **Website**: a static single-page app that talks to Supabase with the *publishable*
  (anon) key only. The build refuses to bundle a `sb_secret_…` or `service_role` key.
  Login is email + password.

## Setup (one time, about 10 minutes)

1. **Create the Supabase project.** In the dashboard, click New project and name it
   `true-adam-podcast`. Use the same organisation as your existing work so the Supabase
   connector can see it, but make it a new project. Free plan note: free projects pause
   after about a week with no traffic, and an account can only have a limited number of
   free projects.
2. **Apply the schema.** Run `supabase/migrations/20261106000000_init.sql` in the SQL
   editor, or with the connector's `apply_migration`, or with `supabase link && supabase db push`.
3. **Auth settings** (Authentication → Sign In / Providers):
   - Turn **off** "Allow new users to sign up". The allow-list blocks strangers anyway, but this keeps junk accounts out.
   - Leave the Email provider on. No SMTP setup is needed, because users are created with passwords.
4. **Add the crew.** Insert each email into `allowed_emails`, then create the users under
   Authentication → Users → Add user, with **Auto Confirm User** ticked.
   [docs/PRODUCER.md](docs/PRODUCER.md#crew-access) has the exact SQL.
5. **Deploy the site** (GitHub Pages, free for this public repo):
   - Settings → Pages → Source: **GitHub Actions**.
   - Settings → Secrets and variables → Actions → **Variables**: add `SUPABASE_URL`
     (`https://<ref>.supabase.co`) and `SUPABASE_PUBLISHABLE_KEY` (Project Settings → API
     Keys → publishable key, or the legacy anon key).
   - Merge to `main`. The `Deploy site` workflow publishes to
     `https://crazymonkey-1470.github.io/TrueAdamPodcast/`.

   Any other static host works too (Netlify, Vercel, Cloudflare Pages). Build `web/` with
   `npm run build`, publish `web/dist`, and set the same two `VITE_*` variables.

Code is public, data is not. The publishable key and project URL are meant to be public.
Every table is protected by RLS, and the allow-list decides who sees anything.

## Local development

```bash
cd web
cp .env.example .env.local   # fill in URL + publishable key
npm install
npm run dev
```

## Tests

```bash
scripts/test-db.sh   # needs Postgres 15+ server binaries; runs as-is on GitHub's ubuntu runners
```

This spins up a throwaway Postgres with a small Supabase stand-in (`auth.users`,
`auth.uid()`, the `anon`/`authenticated` roles, and Supabase's default grants). It applies
the migration and checks 76 behaviours: the allow-list, duplicate URLs, topic validation,
anon lockout, host vs. producer edit rights, attribution, and the views.

## Site map

| Route | What |
|---|---|
| `#/` | Next recording, **This week** pile (flags a missing fun fact or guest), latest inbox |
| `#/t/<slug>` | Topic tab: inbox + this week, used, archived, all |
| `#/i/<id>` | One item with comments (shareable link) |
| `#/add` | Add an item (article, video, note, fun fact, guest idea) |
| `#/episodes`, `#/e/<id>` | Episode ideas: attach items, checklist (date, guest, fun fact, 3+ discussion items), mark all used |
| `#/search/<q>` | Search titles, summaries, sources |
| `#/account` | Display name, password, sign out |

Each card shows the title (linked), source, published date with freshness, summary,
talking points, quote, topics, attached episodes, and the actions: ★ This week, ✓ Used,
Archive, comments, and add to episode.
