# diffy-skills

Claude Code plugin marketplace for [Diffy](https://diffy.website/) visual-regression testing.

This repo distributes a single plugin, [`diffy`](plugins/diffy), built on the `diffy` CLI, with **two
capture modes**:

- **Local** — capture a **running app on your machine** inside Diffy's published
  `diffywebsite2/screenshot-worker` Docker container (the production rendering runtime, run locally) and
  upload it. Requires Docker.
- **Remote** — have **Diffy capture on its own servers** (no local browser or Docker) — screenshot an
  environment or diff two environments server-side.

When a request doesn't say where to capture, Claude asks whether to run **local** or **remote** (on
app.diffy.website). See [`plugins/diffy/README.md`](plugins/diffy/README.md) for the full skill list and
prerequisites.

## Install

```shell
/plugin marketplace add diffywebsite/diffy-skills
/plugin install diffy@diffy
```

Then the skills are available namespaced, e.g. `/diffy:visual-diff` or `/diffy:compare-environments`.
Claude also invokes them automatically based on your request.

## Skills

| Skill | Mode | What it does |
|---|---|---|
| `/diffy:create-project` | either | Create a new Diffy project from a base URL, pages, and breakpoints. |
| `/diffy:get-project-settings` | either | Show an existing project's settings as export YAML. |
| `/diffy:update-project-settings` | either | Edit pages, environments, breakpoints, masks, login, schedule, etc. |
| `/diffy:upload-screenshot` | local | Capture a running app in the screenshot-worker container and upload it. |
| `/diffy:remote-screenshot` | remote | Ask Diffy to screenshot an environment on its servers. |
| `/diffy:compare-screenshots` | either | Create a diff from two existing screenshot set IDs. |
| `/diffy:get-diff-info` | either | Summarize a diff: % changed, per-page/breakpoint table, report link, JUnit. |
| `/diffy:visual-diff` | local | End-to-end before/after regression on your **local** UI. |
| `/diffy:compare-environments` | remote | Compare two environments server-side. |

## Local development / testing (no install)

```bash
# from the repo root
claude --plugin-dir ./plugins/diffy
claude plugin validate ./plugins/diffy    # run before publishing / submitting
```

Use `/reload-plugins` inside a session to pick up edits without restarting.

## License

MIT — see [LICENSE](LICENSE).
