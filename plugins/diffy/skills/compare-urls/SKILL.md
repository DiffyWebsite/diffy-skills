---
name: compare-urls
description: >-
  Guided, end-to-end visual comparison of two live URLs for web developers (optimized for Astro sites):
  sets up a Diffy account and API key if needed, creates a project with URL A as production and URL B as
  staging, lets the user pick breakpoints and pages, runs the comparison on Diffy's servers, waits for it,
  and returns the shareable report link. Use when the user has two URLs (e.g. live site vs preview deploy)
  and wants Diffy to compare them, especially first-time Diffy users or Astro projects. For an existing
  project's configured environments use compare-environments; for a local dev server use visual-diff.
allowed-tools: Bash, Read, Write
---

# compare-urls

One guided conversation: two URLs in, a shareable Diffy report out. This skill orchestrates the same CLI
commands the granular skills use (`create-project`, `update-project-settings`, `compare-environments`,
`get-diff-info`); it does not reimplement them.

**Never delete projects** (or screenshots or diffs), by any means. If the user asks, tell them to do it in
the Diffy dashboard.

## Constants

Keep these in this one place so they can be changed without touching the steps below.

- Coupon: **`CLAUDE2026`**. It is public (this repo is public), so never present it as secret. It turns on
  API access and the custom environments for the account it is applied to.
- Signup attribution: `--location="claude-code?utm_source=claude-code"`
- Feedback source: `--source=claude-code/compare-urls`
- Default breakpoints: `[640, 1024, 1200]` (Diffy's own default)
- Minimum CLI: **0.1.56** (`auth:register`, `auth:status`, `feedback:send`, working `--wait`)
- Dashboard: https://app.diffy.website/#/ · keys: `#/keys` · profile (coupon field): `#/profile`

## 0. Preflight

1. Check PHP. The CLI is a PHP archive, so it needs PHP 8.2+ on `PATH` even when installed as a phar:
   ```bash
   php -r 'exit(PHP_VERSION_ID >= 80200 ? 0 : 1);'
   ```
   If `php` is missing or the check fails, stop and tell the user to install or upgrade PHP first, then
   come back: macOS `brew install php`, Debian/Ubuntu `sudo apt install php-cli` (check that the distro
   ships 8.2+), Windows https://windows.php.net/download/. Don't install it for them. Re-check once
   they say it is done.
2. Resolve the CLI: `diffy` on `PATH`, else `./vendor/bin/diffy`, else ask the user to install it and stop
   until it is available:
   `wget -O /usr/local/bin/diffy https://github.com/diffywebsite/diffy-cli/releases/latest/download/diffy.phar && chmod a+x /usr/local/bin/diffy`
   (pick a bin directory the user can write to; `/usr/local/bin` may need `sudo`, Homebrew on Apple Silicon
   uses `/opt/homebrew/bin`).
3. Probe for the commands this skill needs:
   ```bash
   diffy list --raw | grep -q '^auth:register'
   ```
   If the probe fails, the build is older than 0.1.56. Offer to upgrade by re-running the install one-liner
   to the path `command -v diffy` reports (Composer installs: `composer update diffy-website/diffy-cli`).
   Say what you will run and get the user's go-ahead first; upgrade at most once per session. If the user
   declines or the probe still fails, continue in **legacy mode**: account creation is manual (§2.3
   fallback), and feedback is saved locally only (§7).

## 1. The two URLs

- Ask for URL A and URL B if they weren't given. Unless the user says otherwise, **A is the baseline
  (`production`)** and **B is the change (`staging`)**. Repeat the mapping back in one line.
- Normalize: require `http://` or `https://`, strip trailing slashes, keep any base path
  (`https://example.com/blog`). A and B should point at the same page of the two sites, because every page
  path is resolved against each of them.
- **Local URLs can't be compared here.** If either host is `localhost`, `127.0.0.1`, `0.0.0.0`, `*.local`,
  `*.test`, a private IP (10/8, 172.16/12, 192.168/16) or a name that resolves to one, stop: Diffy captures
  on its own servers and can't reach it. Offer the `visual-diff` skill (captures on this machine) or a
  public preview deploy (`astro build` + Netlify/Vercel/Cloudflare preview).
- Probe both: `curl -sS -o /dev/null -w '%{http_code}' -m 15 -L <URL>`.
  - `401`: the site uses HTTP basic auth. Ask for the username and password for that side (see §5 for how
    they are used). They are passed on the command line for each compare and **never stored**.
  - Vercel/Netlify deployment protection that redirects to a login page isn't basic auth and isn't
    supported; ask the user to make the preview public or use a bypass URL.
  - Other errors: show them and ask whether to continue anyway.
- If the working directory has `astro.config.{mjs,ts,js,cjs,mts}`, note it for page suggestions in §3.

## 2. Account

Run `diffy auth:status` (legacy mode: check that `${DIFFYCLI_CONFIG:-~/.diffy-cli/diffy-cli.yaml}` exists
and `diffy project:list` succeeds).

| Exit code | Meaning | Next |
|---|---|---|
| `0` | Logged in, API works. It prints the account email. | §3 |
| `2` | The account has no API access. | §2.2 |
| `1` | No key saved, or the key is invalid. | Ask: **"Do you already have a Diffy account?"** Yes → §2.1, No → §2.3 |

### 2.1 Existing account: log in with an API key

Never ask for the user's Diffy password.

1. Send them to https://app.diffy.website/#/keys to create a key. If that page says API keys are
   "Available on paid plans", do §2.2 first.
2. Log in with the key. Either they paste it and you run `diffy auth:login <KEY>`, or, to keep the key out
   of the conversation, they type `! diffy auth:login <KEY>` themselves. Don't echo the key back.
3. Exit `0` → §3. Exit `2` → §2.2. Exit `1` → the key is wrong; ask for it again once.

### 2.2 Existing account without API access: apply the coupon in the web app

The skill never redeems the coupon itself. Tell the user:

> Your Diffy account doesn't have API access yet. Open https://app.diffy.website/#/profile, enter the
> coupon **`CLAUDE2026`** in the Coupon field and redeem it. Then create a key at
> https://app.diffy.website/#/keys and give it to me (or run `! diffy auth:login <KEY>`).

Wait for them to confirm, then run `diffy auth:status` (or `auth:login` with the new key).
- `0`: continue to §3.
- Still `2`: say the coupon doesn't seem to be applied yet. Offer one more check, or support@diffy.website.
  **Don't check more than twice.**
- If they don't want to apply it, stop and explain that API access is needed to run the comparison.

### 2.3 No account: create one

1. Find their GitHub email and name:
   ```bash
   gh api user/emails --jq '.[] | select(.primary and .verified) | .email'   # needs gh auth with user scope
   gh api user --jq '.email // empty'
   git config user.email
   gh api user --jq '.name // .login'
   git config user.name
   ```
   Use the first non-empty email (skip `*@users.noreply.github.com`) and the first non-empty name. If you
   find none, ask.
2. **Ask before creating anything** (this creates an account on an external service): "Create a Diffy
   account for `<email>` (name `<name>`)?" Let them change the email or name.
3. Run:
   ```bash
   diffy auth:register --email="<email>" --name="<name>" --coupon=CLAUDE2026 --location="claude-code?utm_source=claude-code"
   ```
   The CLI generates a random password it never prints or stores, registers with the coupon, creates an
   API key and saves it, and emails a link to set a password.
4. On exit `0`, tell the user: "Your Diffy account is ready and this terminal is logged in. We emailed
   `<email>` a link to set your password; you only need it for the web dashboard at app.diffy.website."
   Continue to §3. No email verification is needed.
5. Exit `2`: the account **was created** but the coupon wasn't applied (the output says why), so it has no
   API access. Tell them to set their password from the email, log in, then follow §2.2.
6. Exit `1` with "already exists": they already have an account. Go to §2.1 and offer
   `diffy auth:reset-password <email>` in case they forgot the password.
7. Any other failure: show the CLI's message and fall back to manual signup.

**Legacy mode / manual fallback:** ask them to sign up at https://app.diffy.website, redeem `CLAUDE2026` at
`#/profile`, create a key at `#/keys`, then continue with §2.1 step 2. Say this step is manual because
their `diffy` build is too old.

## 3. Breakpoints and pages

Ask both in one message where you can.

**Breakpoints:** show the defaults `640, 1024, 1200` and let them change them. Accept integers from 320 to
2560, de-duplicate and sort them.

**Pages:** "Just the homepage, or several pages?"
- **One:** `/`, or the single path they name.
- **Several:** accept paths such as `/about-us, /pricing`. Offer suggestions:
  - *Astro routes* (if §1 found an Astro config): list static routes under `src/pages/**` with extensions
    `.astro`, `.md`, `.mdx`, `.html`. `index.astro` → `/`, `about-us.astro` → `/about-us`,
    `blog/index.md` → `/blog`. Skip files and folders starting with `_`, API endpoints (`.js`/`.ts`
    files) and dynamic routes (`[slug]`, `[...rest]`) unless the user gives concrete values. If
    `astro.config` sets `base` and URL A doesn't already end with it, prefix the paths with it; if it sets
    `trailingSlash: 'always'`, add the trailing slash.
  - *Sitemap:* `curl -fsSL -m 15 <A>/sitemap-index.xml` (what `@astrojs/sitemap` generates, follow its
    `<sitemap><loc>` entries) or `<A>/sitemap.xml`; take the `<loc>` paths.
  - Show at most about 25 suggestions and let them pick.

Resolve every path against A to get absolute `urls` (`create-project` rule). Before running, show the
total: pages × breakpoints × 2 sites = screenshots. If it's over 25 pages, confirm.

## 4. Project: reuse or create

1. `diffy project:list` prints JSON. Look for a project whose `production` equals A (ignore a trailing
   slash).
2. **Found:** offer to reuse it instead of creating a duplicate. If they agree, update it with the
   `update-project-settings` flow: `diffy project:get <ID>`, write
   `.diffy-skills/project-settings/<ID>.json` with `staging` = B and the chosen `breakpoints` and `urls`
   (keep the other fields), then `diffy project:update <ID> <file>`.
3. **Not found (or they want a new one):** confirm creating it, then write
   `.diffy-skills/project-create/<safe-name>.json` and run `diffy project:create <file>`:
   ```json
   {
     "name": "<A host> vs <B host>",
     "breakpoints": [640, 1024, 1200],
     "production": "<A>",
     "staging": "<B>",
     "development": "",
     "urls": ["<A>/", "<A>/about-us"]
   }
   ```
   Parse `[<ID>] created.` and verify with `diffy project:get <ID>`.
4. Offer to add `.diffy-skills/` to `.gitignore` if it isn't ignored yet.

## 5. Compare and wait

Start the comparison without `--wait`, so no single command runs longer than a tool call allows:

```bash
DIFF_ID=$(diffy project:compare <ID> prod stage --name="<A host> vs <B host> <YYYY-MM-DD>")
```

**With basic auth** on either side, compare the URLs as custom environments so the credentials go with this
one command and are never saved on the project:

```bash
DIFF_ID=$(diffy project:compare <ID> custom custom \
  --env1Url="<A>" --env2Url="<B>" --env2User="<user>" --env2Pass="<pass>" \
  --name="<A host> vs <B host> <YYYY-MM-DD>")
```

(`--env1User`/`--env1Pass` for A.) Quote the values. Don't write the credentials to any file.

Then poll until it's done, in Bash calls of at most about 9 minutes each (set the tool timeout to 600000):

```bash
for i in $(seq 1 36); do
  STATE=$(diffy diff:get-result "$DIFF_ID" --format=json | jq -r '.state')
  case "$STATE" in 2|3|4|7) echo "completed"; break;; -1) echo "failed"; break;; esac
  sleep 15
done; echo "state=$STATE"
```

(No `jq`? Use `python3 -c 'import sys,json; print(json.load(sys.stdin)["state"])'`.) Between calls, post a
short progress line. Stop after 20 minutes in total: give the user the diff ID and tell them the report
will also arrive by email; `get-diff-info <DIFF_ID>` shows it later. On `-1`, say the comparison failed and
show the diff ID.

Write `.diffy-skills/diffs/last-diff.json` (no credentials):

```json
{"projectId": 12345, "env1": "production", "env2": "staging", "diffId": 333, "name": "…", "waited": true,
 "flow": "compare-urls", "createdAt": "2026-10-25T12:00:00Z"}
```

## 6. Result

From `diffy diff:get-result <DIFF_ID> --format=json` (same parsing as `get-diff-info`):
- Headline: `result`% of pages changed, or "No visual changes found" when it is `0`.
- A compact table of page × breakpoint `percentageChanges` from `diffs`, non-zero rows first.
- The **shareable report link `diffSharedUrl`**, on its own line. Offer to open it (`open` on macOS,
  `xdg-open` on Linux).
- The project ID and diff ID, and that `compare-environments` re-runs the comparison on the same project.

## 7. Feedback

Ask: **"How did you like the comparison?"**, with the choices 👍 Useful / 😐 OK / 👎 Not useful / Skip,
and an optional comment. Say in the question: "Your answer and account email will be sent to the Diffy
team at support@diffy.website." Answering is consent; **Skip** sends nothing. Ask only once.

If they answered:

```bash
diffy feedback:send "$DIFF_ID" --rating=<useful|ok|not_useful> --comment='<comment>' --source=claude-code/compare-urls
```

Pass the comment as one argument, single-quoted with any `'` escaped. Set `DIFFY_PLUGIN_VERSION` to this
plugin's version from its `plugin.json` if you can read it.

Always also append one line to `.diffy-skills/feedback.jsonl`:

```json
{"at": "2026-10-25T12:00:00Z", "diffId": 333, "rating": "useful", "comment": "…", "sent": true}
```

If sending fails, or in legacy mode, set `"sent": false`, tell the user it was saved locally, and move on.
Never let feedback hold up the result.
