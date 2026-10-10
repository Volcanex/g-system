# Installing G-System

This guide is for an AI agent setting up G-System on a fresh Linux server and a Mac. Read it to the end before you run anything, because the bootstrap scripts replace dotfiles in the home directory they run in. They back up what they replace, but test on a throwaway user or VM first if the home directory matters to anyone.

## What you need

- A Linux server with root, systemd, git, Python 3 and Node 22. Mine runs Ubuntu 24.04 with Python 3.12.
- A Mac with Homebrew, git and Node 22, for the desktop app.
- Read access to the `g-` repos on GitHub, and a private repo of your own for the instance.
- An ntfy topic, on ntfy.sh or a server you run. On ntfy.sh the topic name is the secret, so make it long and random.

## Layout

Clone every repo into one directory, side by side, on every host. The resolver finds the instance by looking beside g-env, so the sibling layout is what holds the system together.

```
~/g/
  g-env/          framework: bootstrap, skills, hooks, settings
  g-env-linux/    Linux layer (servers)
  g-core/         schema and renderer (only where you edit the manifest)
  gp-core/        your instance, private
  g-brain/        brain host and its packages (central server)
  g-command-web/  dashboard (central server)
  g-browser/      optional
```

This repo pins every framework repo as a submodule at its root, so one clone gives the same layout:

```sh
git clone --recurse-submodules https://github.com/Volcanex/g-system ~/g
cd ~/g && git submodule foreach git switch -q main
```

The pins are the commits this guide was checked against. Switch each one to main as above, because the daily sync skips a repo on a detached checkout and would never update it. The clone also brings g-design, g-seed and g-command-mac, which a server can ignore. If you install this way, skip the `git clone` lines for `g-` repos in the steps below.

The instance directory must be named `gp-core`, or `gcore` for older hosts, unless you point `G_INSTANCE` at it.

## 1. Make the instance

Do this once, on the machine where you will edit the manifest.

```sh
cd ~/g
git clone https://github.com/Volcanex/g-core
cp -r g-core/examples gp-core
cd gp-core && git init
mkdir .githooks && cp ../g-core/.githooks/pre-commit .githooks/
git config core.hooksPath .githooks
python3 ../g-core/render_context.py --instance . --check
python3 ../g-core/render_context.py --instance .
```

Then fill it in and push it to your private remote. An instance holds:

| Path | Needed | What it holds |
|---|---|---|
| `manifest.json` | yes | Hosts, workspaces, system repos, services, external dependencies and projects. `g-core/manifest.schema.json` lists the keys. |
| `CLAUDE.md` | yes | The topology block, rendered from the manifest. The pre-commit hook renders it again on every commit. |
| `conventions.md` | yes | House rules for agents. The session loader injects the block between its SESSION markers. Start from `g-core/conventions.md`. |
| `.githooks/pre-commit` | yes | The render hook from g-core. |
| `.env` | yes | The secrets passport, `KEY=value` lines. Probes in the manifest name keys and never hold values. |
| `presentation.json` | no | Labels and grouping for the dashboard's system diagram. |
| `env/` | no | Files that replace framework defaults, from the overlay table below. |
| `skills/` | no | Skills for this instance only. They load beside the framework's skills. |

I track `.env` in my private instance so every host gets it on pull. If you would rather not, leave it untracked and copy it to each host yourself. Either way the instance must never be public.

## 2. How g-env finds the instance

`bin/g-instance` resolves the instance in this order and stops at the first hit:

1. A file-specific legacy variable: `GCORE_ENV`, `GCORE_MANIFEST`, `GCORE_DIR` or `GSITE_GCORE_DIR`. These are explicit operator settings, so they win.
2. `G_INSTANCE`, an absolute path.
3. `<parent of g-env>/gp-core`, if it exists.
4. `<parent of g-env>/gcore`, the legacy name.
5. Nothing found: it prints how to make one from `g-core/examples` and the framework defaults still install.

`g-instance` prints the directory, `g-instance env` prints the passport path and `g-instance manifest` prints the manifest path. The framework checkout is `G_ENV`, then `GENV`, then `GENV_DIR`, then the script's own repo. If `G_INSTANCE_REMOTE` is set and nothing resolves, bootstrap clones that remote into `<parent of g-env>/gp-core`.

The overlay table links the instance's copy of a file if `$G_INSTANCE/env/<path>` exists, and the framework default if not. It replaces the whole file and never merges.

| Framework default | Instance override | Target |
|---|---|---|
| `g-env/claude/settings.json` | `env/claude/settings.json` | `~/.claude/settings.json`, replaced only if absent or already a link bootstrap made |
| `g-env/git/gitconfig` | `env/git/gitconfig` | `~/.gitconfig` |
| `g-env/codex/config.toml` | `env/codex/config.toml` | `~/.codex/config.toml`, on the Mac only and only from an instance copy |
| `g-env-linux/ssh/config` | `env/ssh/config` | `~/.ssh/config` |

Instance settings refer to hooks only through `~/.claude/hooks/<name>`, which the framework guarantees. When the framework adds a hook, its CLAUDE.md says so, and you add the hook to your own settings.

## 3. Bootstrap each Linux host

As root on each server:

```sh
cd ~/g
git clone <your instance remote> gp-core
git clone https://github.com/Volcanex/g-env-linux
bash g-env-linux/bootstrap.sh
```

`bootstrap.sh` clones g-env beside itself if it is missing and runs `g-env/bootstrap-base.sh`. That installs the shared context and skill library under `~/.agents`, Claude Code's agents, hooks and settings, the Codex rules and hooks, the helper commands in `~/.local/bin` and the git config. The Linux layer then links the shell and SSH config, creates `~/.claude/settings.local.json` from the Linux defaults, installs `genv-sync.timer` and adds a root cron entry that mirrors Claude config every five minutes. The timer pulls g-env daily and runs the bootstrap again, so a push to g-env reaches every server overnight.

The script reads `G_ENV` (falling back to `GENV_DIR`), uses the resolver above, and keeps the old names of the `genv-sync` timer and the `gsync` command so existing hosts carry on.

## 4. Run the services on the central server

g-brain ships example units for its rows in `deploy/systemd/`, with `g-alert@.service` as the failure pager, and g-browser ships its own in `deploy/`. Fill in their `@NAME@` placeholders, copy them to `/etc/systemd/system/` and enable them. The dashboard has no unit yet, so write one for it. Keep the ports on 127.0.0.1 and put a reverse proxy in front of anything a browser reaches.

| Service | Repo | Command | Needs |
|---|---|---|---|
| Brain host | g-brain | `.venv/bin/uvicorn brain:app --host 127.0.0.1 --port 8766` | `BRAIN_ROOT`, a git repo of markdown notes. basic-memory serving that vault on port 8765. One process serves `/mcp` to agents and `/api` to the web. |
| Status probes | g-brain (g-spine) | `python brain_status.py`, every 5 minutes | `NTFY_URL` in the passport. Add `OnFailure=g-alert@%n.service` to your other units so a failed job alerts too. |
| Reminders | g-brain (g-todos) | `python -m outlets.remind`, every 5 minutes | `NTFY_URL` |
| Voice assistant | g-brain (g-voice) | `uvicorn voice.app:app --host 127.0.0.1 --port 8077` | `voice/.env`, filled from `voice/.env.example`. Optional. |
| Dashboard | g-command-web | `python3 compile.py && python3 server.py` | `PORT` (default 8089), `ADMIN_PASSWORD`, and `ADMIN_COOKIE_VALUE`, without which it refuses to start. `BRAIN_API`, the origin of your brain's `/api`, which `compile.py` bakes into the pages. |
| Browser | g-browser | the units in `deploy/` | `GBROWSER_PASSWORD`, `GBROWSER_SERVICE_TOKEN`, `GBROWSER_COOKIE_VALUE`. Optional. |

`/mcp` needs to reach your agents' clients. I expose it through a tunnel with OAuth in front, and the brain handles the OAuth itself. `/api` goes through the reverse proxy.

Still moving here:

- The g-brain package split keeps `brain:app`, `brain_status` and `brain_alerts` working as aliasing shims, so these commands do not change when it lands.
- Several g-brain defaults still name the original owner's paths. Set `BRAIN_ROOT` and `GCORE_MANIFEST` explicitly rather than trusting the defaults.
- The g-command-web calendar page still fetches a fixed brain API address. Edit `pages/calendar/content.html` to point at your brain host's `/api` until it reads `BRAIN_API` like the other pages.

New web apps start from g-seed. Its README has the steps.

## 5. Set up the Mac

The Mac runs the desktop app. There is macOS support for the agent environment too, but I'm still cutting it out of my own Mac setup, so this guide does not cover it yet.

```sh
git clone https://github.com/Volcanex/g-command-mac
cd g-command-mac && npm install && npm test && npm start
```

Sign in to the dashboard in the app window. Open File, Open settings file, set `ntfy.topics` to your topic (the path of `NTFY_URL`) and reload settings. The CLAUDE.md in g-command-mac covers signing and notarising.

## 6. Verify

Check each of these and report what you saw, not what you expected.

1. `g-instance`, `g-instance env` and `g-instance manifest` print paths inside your instance.
2. `python3 g-core/render_context.py --instance <instance> --check` exits 0.
3. `ls -l ~/.claude/settings.json ~/.gitconfig ~/.ssh/config` shows each link pointing where the overlay table says.
4. `python3 ~/.agents/session-context.py` prints a "G-System session context" block with your hosts and your conventions in it.
5. A new Claude Code session starts with that same block injected, and `ls ~/.claude/skills/` lists the framework skills and your instance skills.
6. `systemctl list-timers genv-sync.timer` shows the next run.
7. `curl -s http://127.0.0.1:8766/api/status` returns the status snapshot, and the services in it match your manifest.
8. `/mcp` answers through your tunnel, and an agent session can call the brain's tools.
9. The dashboard's `/system` page lists your hosts and services after you sign in.
10. `python -m brain_alerts --level info --source Reminder test`, run from the g-brain checkout, arrives on your phone and in the desktop app, and clicking it opens the calendar.
11. The desktop app's menu-bar panel shows system health and today's todos, and its alerts row reads On.
