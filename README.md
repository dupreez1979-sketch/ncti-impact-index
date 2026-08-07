# Children's Theatre Impact Index

A live modelling dashboard for the National Children's Theatre Initiative. It scores
every Australian SA2 for children's-theatre impact (AEDC emotional maturity, SEIFA
disadvantage, remoteness), maps venues against it, and models touring subsidy.

Single static page (`index.html`). Data is served live from Supabase (project
`ezvpiobtzftmrqxdnebh`) — nothing is baked into the page.

## Deploy (Netlify from GitHub)
1. Push this folder to a GitHub repo (see below).
2. In Netlify: **Add new site → Import an existing project → GitHub →** pick the repo.
3. Build settings: **Build command** = *(empty)*, **Publish directory** = `.`
4. Deploy. Every push to `main` redeploys automatically.

## Push to GitHub (command line)
```bash
cd ncti-impact-index
git init
git add .
git commit -m "NCTI Impact Index dashboard"
git branch -M main
git remote add origin https://github.com/<your-username>/ncti-impact-index.git
git push -u origin main
```
No command line? On github.com create a new repo, then **Add file → Upload files** and
drag `index.html`, `netlify.toml`, `.gitignore` and the `supabase/` folder in, and commit.

## Supabase
`supabase/schema.sql` is the database schema (tables, `venue_full` view, row-level
security). The boundary geometry is served from Supabase Storage (bucket `geo`,
`sa2_boundaries.geojson`). The dashboard reads with the project's **publishable** key,
which is safe in the browser because reads are governed by the RLS policies.

## After it's hosted
Editing venues uses Supabase Auth (magic link). In Supabase → Authentication → URL
Configuration, set the Netlify URL as the Site URL and an allowed redirect. The
"Sign in to edit" flow and inline venue editor are added on top of this page.
