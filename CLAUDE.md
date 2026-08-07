# NCTI Impact Index — working notes

Single static page. `index.html` is the whole dashboard; data comes live from
Supabase at runtime. `netlify.toml` publishes the repo root with no build step.

Netlify redeploys automatically on every push to `main`, so pushing **is**
deploying. There is no separate deploy step.

## "update" / "commit" / "push it up"

When Kevin says any of these in this folder, do all of it without asking:

1. `git status` and `git diff --stat` — see what actually changed.
2. Sanity-check `index.html` if it changed: it must still open with `<!doctype html>`
   and the file should not have shrunk dramatically (a truncated save is the main
   risk here — the file is ~150KB).
3. `git pull --rebase origin main` — pick up anything changed on GitHub first.
4. `git add -A` — this stages deletions and renames too, not just edits.
5. Commit with a message describing the actual change, not "update files".
6. `git push origin main`.
7. Report back: what changed, the commit hash, and that Netlify is rebuilding.

If the pull in step 3 conflicts, stop and show Kevin the conflict rather than
guessing which side wins.

## Don't commit

Secrets belong in Supabase or Netlify environment settings, never in `index.html`.
The Supabase **publishable** key is fine in the page (RLS governs reads); the
service-role key must never appear in this repo.
