---
name: get-screenshot-info
description: >-
  Check whether a Diffy screenshot set has finished capturing with screenshot:get-status, and list recent
  screenshot sets for a project. Use when the user asks if a screenshot is done, ready, finished, or still
  running, or asks for screenshot status, screenshot set IDs, or the latest screenshot of a project.
allowed-tools: Bash, Read, Write
---

# get-screenshot-info

Read and report Diffy screenshot set status only. Do not create screenshots, set baselines, create diffs,
or change project settings in this skill.

## Preflight

1. Resolve the CLI:
   - Prefer `diffy` from `PATH`.
   - Else, if `./vendor/bin/diffy` exists in the current repo, use that.
   - Else, ask the user to install it and stop until it is available. Suggested install (pick a bin
     directory the user can write to — `/usr/local/bin` may need `sudo`; Homebrew on Apple Silicon uses
     `/opt/homebrew/bin`):
     `wget -O /usr/local/bin/diffy https://github.com/diffywebsite/diffy-cli/releases/latest/download/diffy.phar && chmod a+x /usr/local/bin/diffy`
2. Check authentication: `~/.diffy-cli/diffy-cli.yaml` must exist. If it does not, run
   `diffy auth:login <API_KEY>` after the user provides the key (key from https://app.diffy.website/#/keys).
3. Check the CLI supports this skill — `screenshot:get-status` landed in **0.1.55**. Probe for the command
   itself rather than parsing `diffy --version`:
   ```bash
   diffy list --raw | grep -q '^screenshot:get-status'
   ```
   Needs no API key, so it is safe to run before auth. If it exits non-zero the CLI is too old: tell the
   user they need `diffy` 0.1.55 or newer and to reinstall the phar with the one-liner above — it always
   fetches the latest release — then stop until they have. This CLI has **no `self:update` command**;
   reinstalling the phar is the only upgrade path. Users on `./vendor/bin/diffy` upgrade with
   `composer update diffy-website/diffy-cli` instead.

   Do not gate on `diffy --version`. Released phars have historically reported a stale version string
   (every build from 0.1.50 through 0.1.54 reports `DiffyCli 0.1.49`), so the command list is the only
   trustworthy signal.

## Resolve the Screenshot

Preferred input: `SCREENSHOT_ID`.

If the user does not give one, look for it in this order before asking:

1. Metadata this plugin already wrote — `.diffy-skills/screenshots/*.json` (from `remote-screenshot`) and
   `.diffy-skills/uploads/*/metadata.json` (from `upload-screenshot`). Both carry a `screenshotId` field.
2. Recent sets on the project, if a `PROJECT_ID` is known:

   ```bash
   diffy screenshot:list <PROJECT_ID> --limit=10
   ```

   This prints a PHP `var_export` dump, not JSON. Read each set's ID from its `'id' => <number>` line
   rather than trying to parse the output as JSON. The dump also carries `name`, `environment`, and `date`,
   which are useful for reporting, but it does **not** include a state field — status comes only from
   `screenshot:get-status` below.

Use the most recent set only when the user clearly asked for the latest one. Otherwise ask which
`SCREENSHOT_ID` to check.

## Check Status

Always use the JSON form — it is the only output that says *why* a set is not complete:

```bash
diffy screenshot:get-status <SCREENSHOT_ID> --format=json
```

Fields in the output:
- `completed` — boolean; `true` for states 2, 3, and 4.
- `state` — numeric capture state. Map it for the user:
  - `-1` — **stopped**: the capture did not finish. Tell the user it failed; do not present it as pending.
  - `0` — not started (queued).
  - `1` — in progress.
  - `2`, `3`, `4` — completed (`3` = completion hook fired, `4` = zip ready).
- `status` — live progress: `results` of `items` screenshots done, plus `queue`, `jobs`, `inProgress` and
  a human `estimate` string such as `"under 1 minute"`. Report progress as `results/items`.
- `environment`, `name`, `executionTime` — labels for the summary. `executionTime` (e.g. `"2min 6s"`) is
  the total capture time once finished.

Note that `state` and the bare output mean different things: the plain
`diffy screenshot:get-status <SCREENSHOT_ID>` (no `--format`) prints `1` for **completed** and nothing
otherwise, while `state: 1` in the JSON means **in progress**. Only use the bare form in a shell
conditional; use `--format=json` whenever you report to the user.

Exit code is `0` whether or not the set is complete, so branch on the output, not on the status code.

If the command errors with `Command "screenshot:get-status" is not defined`, or with
`Format value is not supported`, the installed `diffy` CLI predates 0.1.55 — the Preflight probe should
have caught this. Tell the user to reinstall the phar (see Preflight) and stop; do not fall back to
guessing status from `screenshot:list`, which carries no state field.

## Waiting

Only poll when the user asks to wait for completion. Re-run the `--format=json` command every 10 seconds
and cap the attempts (120 by default, matching the CLI's own `--max-wait=1200`). Stop early when `state`
is `-1` — that set will never complete. If `completed` is still false at the cap, stop and report how long
you waited, the last `state`, and the last `results/items` progress. Never poll without a cap.

If the user instead wants to block during capture in the first place, that is
`screenshot:create --wait` in the `remote-screenshot` skill, not this one.

## Output

Write status to `.diffy-skills/screenshots/status-<SCREENSHOT_ID>.json`:

```json
{
  "screenshotId": 67890,
  "projectId": 12345,
  "complete": true,
  "state": 3,
  "progress": "84/84",
  "environment": "Production",
  "executionTime": "2min 6s",
  "checkedAt": "2026-07-06T12:00:00Z"
}
```

Return a one-line verdict, then the supporting detail:
- complete — `Complete — 84 screenshots in 2min 6s`
- in progress — `In progress — 41/84 screenshots, est. under 1 minute`
- queued — `Queued — capture has not started yet`
- stopped — `Stopped (state -1) — the capture did not finish`

Plus screenshot ID, project ID and set name when known, and how long you polled if you waited.

If the set is complete, tell the user to use `compare-screenshots` to diff it against another screenshot
ID. If it is not complete, tell them to re-run this same skill later. Do not start a new capture unless
the user explicitly asks for one.

This skill writes metadata under `.diffy-skills/`; offer to add `.diffy-skills/` to the repo's `.gitignore`
if it is not already ignored.
