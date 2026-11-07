# True Adam Podcast: prep site

A private, shared newsroom for Freddy, Stew and Mike for the week between episodes. It has
topic tabs, saved articles with summaries and talking points, a "this week" pile, episode
planning, and comments. The producer fills it from the database side. The hosts browse,
add their own finds, comment, and mark items used.

**Look:** a Renaissance reading room. Plaster ground, fresco pigments (lapis, vermilion,
terre verte, old gold), gilt frames, Roman capitals (Cinzel) for labels and buttons, and
Garamond for reading (Cormorant Garamond for titles, EB Garamond for body). Summaries
open with a red drop cap, and talking points carry red Roman numerals. The Home and
sign-in pages are headed by Michelangelo's *Creation of Adam* (public domain, Wikimedia
Commons): the hands on phones, both figures on wider screens.

```
supabase/migrations/   schema, RLS, seed topics: the contract the producer writes against
supabase/tests/        schema + RLS tests (run on plain Postgres via scripts/test-db.sh)
docs/PRODUCER.md       how to insert items, plus the exact table/column list
web/                   the site: Vite + React + supabase-js, static, mobile-first
.github/workflows/     CI (DB tests + build) and GitHub Pages deploy
```

## How it fits together

- **Database**: the dedicated Supabase project **`TrueAdamPodcast`** (ref `olzqrpbwayzqyvjeotvz`, us-west-2), separate from GovContractHunter.
  RLS is on for every table. The anon role gets nothing. Signed-in users get access only
  if their email is in `allowed_emails`. Hosts can't rewrite text they didn't add.
- **Producer**: writes with SQL through the Supabase connector or the SQL editor. That
  bypasses RLS, so no website login is needed. See [docs/PRODUCER.md](docs/PRODUCER.md).
- **Website**: a static single-page app that talks to Supabase with the *publishable*
  (anon) key only. The build refuses to bundle a `sb_secret_…` or `service_role` key.
  Login is email + password.

## Setup status

Already done on the live project (2026-11-06):

- [x] Schema applied: `supabase/migrations/20261106215937_init.sql`. The filename matches
      the remote migration history, so `supabase link --project-ref olzqrpbwayzqyvjeotvz && supabase db push`
      only applies future migrations.
- [x] Verified live with a rolled-back smoke test: allow-list → profile, duplicate URL
      error, anon locked out, non-member sees nothing, host can change status but not
      producer text, comment attribution.
- [x] Auth: public sign-ups **disabled**, minimum password length 8, site URL
      `https://crazymonkey-1470.github.io/TrueAdamPodcast/`. There is no custom SMTP, so the built-in
      mailer allows only about 2 emails/hour. That's why login is email + password, not magic links.
- [x] `web/.env.production` holds the project URL and **publishable** key, so the build needs no secrets.

Still to do (needs a person):

1. **Add the crew.** Insert each email into `allowed_emails`, then create each user under
   Authentication → Users → Add user → Create new user, with **Auto Confirm User** ticked.
   [docs/PRODUCER.md](docs/PRODUCER.md#crew-access) has the exact SQL.
2. **Turn on the site** (GitHub Pages, free for this public repo):
   - Settings → Pages → Build and deployment → Source: **GitHub Actions**.
   - Actions → **Deploy site** → Run workflow. After that, every push to `main` that touches `web/` redeploys.
   - The site is served at `https://crazymonkey-1470.github.io/TrueAdamPodcast/`.

   Any other static host works too (Netlify, Vercel, Cloudflare Pages): build `web/` with
   `npm run build` and publish `web/dist`.

If the project is on Supabase's Free plan, it pauses after about a week with no traffic.
Weekly use keeps it awake; if it does pause, restore it from the dashboard.

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
the migration and checks 134 behaviours: the allow-list, duplicate URLs, topic validation,
anon lockout, host vs. producer edit rights, attribution, highlights, the Grok research
queue, Grok's memory (dedupe, ranked recall, revisions, privacy), and the views.

## Site map

| Route | What |
|---|---|
| `#/` | Next recording, **This week** pile (flags a missing fun fact or guest), latest inbox |
| `#/t/<slug>` | Topic tab: inbox + this week, used, archived, all |
| `#/i/<id>` | One item with comments (shareable link) |
| `#/add` | Add an item (article, video, note, fun fact, guest idea) |
| `#/episodes`, `#/e/<id>` | Episode ideas: attach items, checklist (date, guest, fun fact, 3+ discussion items), mark all used |
| `#/picks` | ★ Crew picks (most-starred first) and topic pitches |
| `#/research` | Grok research queue: ask, copy prompt for Grok, paste answers |
| `#/search/<q>` | Search titles, summaries, sources |
| `#/account` | Display name, password, sign out |

Each card shows the title (linked), source, published date with freshness, summary,
talking points, quote, topics, attached episodes, and the actions: ★ This week, ✓ Used,
Archive, comments, and add to episode.
