# ErrorFared

Watches [Secret Flying's Europe deals page](https://www.secretflying.com/europe-flight-deals/)
and **emails you a link** whenever a new post with **ERROR FARE** in the title appears.

- No dependencies (Node 18+ only; uses built-in `curl` to bypass Cloudflare bot-blocking)
- Remembers what it already sent (`seen.json`), so you're emailed each deal once
- Runs on a schedule (every 15 min via GitHub Actions, free on a public repo)

## 1. Get a free email API key (Resend)

1. Sign up at <https://resend.com> (free: 3,000 emails/month).
2. Create an API key at <https://resend.com/api-keys>.
3. You can send **to your own signup email** using the default sender
   `onboarding@resend.dev` — no domain setup needed.

## 2. Set your config (environment variables)

Required:

- `RESEND_API_KEY` — your Resend key
- `TO_EMAIL` — where alerts go (your email)

Optional:

- `FROM_EMAIL` — default `onboarding@resend.dev`
- `PAGE_URL` — default the Europe deals page
- `MATCH` — phrase to match in titles, default `ERROR FARE`

## 3. Test it

```powershell
# See what the script finds right now, without sending email:
node watch.mjs --dry-run
```

Then do a real first run (this seeds `seen.json` with current posts and does
**not** email you the backlog):

```powershell
$env:RESEND_API_KEY="re_xxx"; $env:TO_EMAIL="you@example.com"; node watch.mjs
```

From then on, each run emails only *new* ERROR FARE posts.

## 4. Run it 24/7 for free on GitHub Actions (recommended)

The included workflow ([.github/workflows/watch.yml](.github/workflows/watch.yml))
checks every 15 minutes on GitHub's servers — your PC can be off.

1. Create a **public** GitHub repo (public = unlimited free Actions minutes)
   and push this folder to it:

   ```powershell
   git init
   git add .
   git commit -m "ErrorFared watcher"
   git branch -M main
   git remote add origin https://github.com/YOURNAME/errorfared.git
   git push -u origin main
   ```

2. In the repo: **Settings → Secrets and variables → Actions → New repository
   secret**, add:
   - `RESEND_API_KEY` = your Resend key
   - `TO_EMAIL` = your email

3. Go to the **Actions** tab, open "Watch Secret Flying error fares", click
   **Run workflow** once. The first run seeds `seen.json` (no email); every
   run after that emails you only new ERROR FARE posts.

Notes:
- GitHub can delay scheduled runs by a few minutes — expect roughly every
  15–25 min in practice.
- GitHub disables schedules after ~60 days without repo activity, but the
  workflow's own `seen.json` commits normally keep it active. If you ever get
  a "workflow disabled" email, one click re-enables it.
- The state file `seen.json` is committed back to the repo by the workflow —
  that's expected.

## 4b. Alternative: run on your PC (Windows Task Scheduler)

If you'd rather run it locally, create a task that runs every 15 minutes:

```powershell
$node = (Get-Command node).Source
$dir  = "C:\Users\matij\Desktop\Errorfared"

$action  = New-ScheduledTaskAction -Execute $node -Argument "watch.mjs" -WorkingDirectory $dir
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) `
             -RepetitionInterval (New-TimeSpan -Minutes 15)
# Store your secrets so the task has them (edits your user environment):
[Environment]::SetEnvironmentVariable("RESEND_API_KEY","re_xxx","User")
[Environment]::SetEnvironmentVariable("TO_EMAIL","you@example.com","User")

Register-ScheduledTask -TaskName "ErrorFared" -Action $action -Trigger $trigger `
  -Description "Email me new Secret Flying ERROR FARE deals"
```

Downside: only runs while the PC is on.

## How the bot-block bypass works

Secret Flying is behind Cloudflare. Node's built-in `fetch` gets `403`
(its network fingerprint reads as a bot), but a normal `curl` request with a
browser User-Agent gets `200`. The script shells out to `curl`; if that ever
fails it falls back to the free `r.jina.ai` reader proxy.

## Files

- `watch.mjs` — the whole watcher
- `seen.json` — auto-created; the posts already emailed (delete to reset)
