---
name: offload-to-hetzner
argument-hint: "<what to run off the Mac>"
description: >
  Offload heavy builds, Docker work, test suites and parallel experiments from the local Mac to short-lived
  Hetzner Cloud servers with the `offload` wrapper (create, push, exec --detach, wait, get, rm, clean). Use when a
  task would need several GB of RAM or most CPU cores (Docker, VMs, test matrices, big builds, indexing), when
  several agents compete for one machine, or when a suite should be sharded across machines. A Hetzner alternative
  to Vercel's offload-to-sandbox: every server gets its own SSH key, labels, an expiry and a watchdog, and a cx53
  (16 cores, 32 GB) costs about 0.05 EUR per hour.
metadata:
  version: "1.0.0"
  homepage: https://github.com/pipamann/offload-to-hetzner
---

# Offloading work to a Hetzner Cloud server

Heavy work goes to a server that exists only for the task: create it, push the code, run the job detached, fetch
the results, delete it. One wrapper does all of it: the file `offload` in this skill's folder (the folder this
SKILL.md is in; Claude Code shows it as the skill's base directory). Set it once per shell:

```bash
O=<this skill's folder>/offload
```

`$O --help` lists every subcommand. It needs macOS, Python 3.11+, the `hcloud` CLI (`brew install hcloud`) and a
Hetzner Cloud API token with read & write access. The token is found in this order: the variable `$HCLOUD_TOKEN`
(or the one named in `$OFFLOAD_TOKEN_VAR`), the same name in the dotenv files listed in `$OFFLOAD_SECRET_FILES`
(colon-separated), then hcloud's active context (`hcloud context create <name>`). It only ever travels as an
environment variable.

Before creating a server, propose it to the user in one line with type, expected time and cost, for example: "Test
matrix on a cx53 (16 cores, 32 GB), about 1 h, about 0.05 EUR." Servers cost money until they are deleted.

## When to offload, and what stays local

Offload: Docker builds and stacks, VMs, test matrices and full suites, long builds and anything that needs Linux,
indexing, anything that fills RAM or disk, and several independent tasks at once (one server each).

Keep local: editing, git, quick focused tests, lint and type checks, orchestration, macOS-only tools, anything
interactive, and anything that needs data that must not leave the machine (next section).

## What may go to the server

Secrets and customer data stay local. Upload code, sanitized exports and harmless test data only.

Every upload (`push`, `put`, a `--setup` script) and every command line passes a scan first, and nothing is sent
when it finds something:

- the values of all environment variables with secret-like names (`*TOKEN*`, `*KEY*`, `*SECRET*`, `*PASS*` ...,
  from 8 characters, numbers included), every assignment in the files of `$OFFLOAD_SECRET_FILES`, and the
  project's own `.env*` files (in the folder, the current folder, and up to three levels below the git root);
- common token formats (private key blocks, GitHub, GitLab, Slack, AWS, Stripe live, Google, `sk-` API keys);
- file names that suggest credentials (`.env.production`, `*.pem`, `*.key`, `id_ed25519`, `credentials.json`,
  `*.tfstate` ...) or data dumps (`*.sql`, `*.dump`, `*.sqlite`, `*.db`, `*.bak`).

The scan prints the file and the variable name, never the value. It cannot see secrets that are neither known
locally nor in a known format, and it cannot recognise customer data in ordinary files: that needs your judgment.
For a false positive, check the file, then pass `--allow <path or glob>` (repeatable).

Commands, `--cwd` and remote paths are scanned too, because your shell expands `"$VAR"` before `offload` sees it.
Write remote commands in single quotes. `put` of a symlink and a `--setup` symlink are scanned and sent as their
target; inside `push` and folder `put`, links travel as links and their targets stay home.

## Typical session

Once per Mac: `$O sweeper install` (see "Lifetime, safety net and cleanup").

```bash
$O create parser-tests --type cx53 --setup docker   # ~1 min plain, ~3 min with Docker; prints IP, price, expiry
$O push parser-tests .                              # -> /root/work/<folder name>
$O exec parser-tests --detach suite --cwd /root/work/myrepo -- 'mkdir -p /root/results && make test > /root/results/test.txt'
$O wait parser-tests suite                          # in Claude Code: run_in_background, don't poll
$O get parser-tests /root/results ./offload-results
$O rm parser-tests                                  # deletes server and key, checks both lists, prints the cost
```

Task names are 1-40 lowercase letters, digits and dashes; the server is called `offload-<task>`. Name tasks by
owner and purpose so parallel agents never collide, and never use another agent's server.

## Sizing and prices

Hetzner bills every started hour, even for a five-minute server, and keeps billing a server that is powered off
until it is deleted. `$O types` prints the live prices of every type on sale. Prices on 2026-11-09 in fsn1, without
VAT:

| Type | Cores | RAM | EUR/h | Use |
|---|---|---|---|---|
| cx23 | 2 shared | 4 GB | 0.0088 | smoke tests, tiny jobs |
| cx33 | 4 shared | 8 GB | 0.0136 | one container stack |
| cx43 | 8 shared | 16 GB | 0.0256 | mid-size builds |
| cx53 | 16 shared | 32 GB | 0.0473 | **default**: most cores per euro, for builds, suites and Docker |
| ccx33 | 8 dedicated | 32 GB | 0.2219 | steady timing (benchmarks, timing-sensitive tests) |
| ccx43 | 16 dedicated | 64 GB | 0.4423 | steady timing with lots of RAM |
| cax41 | 16 Arm | 32 GB | 0.0657 | only for Arm builds |

- Shared cores can be slowed by neighbours; dedicated cores are not. For throughput, cx53 beats ccx33 at a fifth of
  the price. Pick ccx33 only when timings must be comparable between runs, and compare timings on the same server.
- Hetzner limits (shared vCPUs, dedicated vCPUs, primary IPs, servers) are per customer account and count across all
  its projects. New accounts start low, for example 8 dedicated vCPUs. When Hetzner refuses a server because of a
  limit, `create` says so; ask the user to request more in the Hetzner Console (Limits, "Request limit increase").
- Offload servers get an IPv4 address only (`--without-ipv6`), so each one uses one primary IP instead of two. SSH,
  apt, GitHub, Docker Hub, PyPI and npm all work over IPv4.
- Without `--location`, `create` tries fsn1, then nbg1, then hel1 when a type is sold out.
- Memory: a build killed with exit status 137 was usually killed by the out-of-memory killer. Take a type with
  more RAM or lower the build's parallelism instead of retrying.

## Moving files

- `push <task> [<local-dir>] [<remote-dir>]` sends what `git ls-files --cached --others --exclude-standard` lists,
  so ignored files such as `.env`, `node_modules` and build output stay home. A staged file is enough; no commit is
  needed. On the server, the new tree replaces the old one and keeps `node_modules` and `.venv` folders from it (up
  to four levels deep; change the list with `OFFLOAD_KEEP`), so dependencies are not reinstalled after every push.
  **Everything else in the old tree is deleted**, including results written there: write results outside the pushed
  folder (for example `/root/results`), or `get` them before the next push. The remote folder must sit below
  `/root`, `/srv`, `/opt`, `/tmp`, `/mnt` or `/home/<user>`, with no symlink on the way; the default is
  `/root/work/<local folder name>` (spaces and other characters become dashes). Submodules are not pushed. Push
  after each change; it takes seconds. If a push fails midway, push again.
- `put <task> <local-file> <remote-file>` copies one file and keeps its mode; `put <task> <local-dir> <remote-dir>`
  copies a folder's files (ignored ones included) into the remote folder.
- `get <task> <remote-path> <local-path>`: a remote folder is merged into the local folder (same names are
  replaced, other local files stay); a remote file goes to the local path, or into it when that is a folder. The
  server ran foreign code, so `get` treats everything it sends as untrusted: only plain files and folders below the
  requested name come back, never links or device files; it never writes through a symlink at or below the local
  path; it refuses `.git` in any spelling, git's control files (`.gitattributes`, `.gitmodules`, `.lfsconfig`,
  `HEAD`, `commondir`, `gitdir`) and anything that would plant or complete a git repository; it refuses other
  hidden files unless you pass `--hidden`; it removes execute bits; and it stops above 20 GB
  (`OFFLOAD_GET_MAX_GB`). Have the job write compact results (a summary, a JSON report, the failing test names)
  and fetch those, not gigabytes of logs.
- Build platform-specific artifacts on the server; never copy macOS binaries there.

## Running commands

- `exec <task> [--cwd <dir>] -- '<command>'` runs now and exits with the remote exit status. Everything after
  `--` is the command; pass it as one single-quoted string so your local shell expands nothing (zsh, for one,
  fails on globs that match no local file). If `--cwd` does not exist, nothing runs. Exit 255 usually means SSH
  failed, in which case the command may or may not have run; a command that itself exits 255 looks the same.
- For anything longer than a few minutes: `exec <task> --detach <job> [--cwd <dir>] -- '<command>'`. The job runs
  under `setsid` on the server and keeps going when SSH drops or the Mac sleeps. Output goes to
  `/root/offload-jobs/<job>.log`, the exit status to `<job>.exit`.
- `wait <task> <job> [--timeout 3h]` polls every 15 seconds, survives dropped connections, prints a progress line
  every 5 minutes and the last 30 log lines at the end, and exits with the job's status. It stops before the server
  expires (15 minutes before, or a quarter of a short lifetime; exit 124), so `get` still has time. In Claude Code,
  run it with `run_in_background: true`.
- `logs <task> <job> [-n 40] [-f]` shows the log; `-f` follows it until the job ends.
- Multi-line work is easier as a script: write it locally, `put` it, and run it. Easier to quote, review and rerun.

## Base snapshot (optional)

`create --setup docker` installs Docker, compose, buildx, git, build-essential, python3 and jq on a fresh Ubuntu
24.04 (about 2 minutes). `--setup <file>` runs your own script instead (it is scanned first; start it with
`set -e`). When many servers need the same toolchain, install it once into a snapshot:

```bash
$O base create docker                            # builder on cx23, default setup "docker", then snapshot, builder deleted
$O base create node22 --setup ./setup-node22.sh  # your own toolchain
$O create shard-1 --type cx53 --from node22      # starts in ~1 min with everything installed
$O base ls                                       # size and monthly cost
$O base rm node22
```

- Snapshots are billed while they exist: 0.0143 EUR per GB and month on the compressed size. The default Docker
  base measured 1.2 GB (about 0.02 EUR a month); `base create` took 2 minutes and a server from it was ready in
  45 seconds. A new `base create` with the same name deletes the older snapshots of that name.
- A snapshot carries the packages of the day it was made, without later security updates. Rebuild it when the
  toolchain changes and at least monthly, and delete bases nobody uses.
- A snapshot only fits types with the same architecture and at least its disk size: a base built on cx23 (x86,
  40 GB) fits every x86 type except cpx02.

## Lifetime, safety net and cleanup

Hetzner has no automatic expiry, so `offload` brings its own, in three layers:

- `--ttl` (default `4h`, between `10m` and `24h`) is the server's lifetime. Set it for the whole task: at expiry the
  watchdog deletes server and key, running jobs included. It is also written into the `offload-expires` label.
- **Watchdog:** a process on this Mac, started before anything is created at Hetzner (`create` refuses to go on if
  it did not start). It checks the wall clock every 30 seconds, so a Mac that slept past the expiry deletes within
  a minute of waking, keeps retrying until both lists are empty (for up to 48 hours after the expiry), and holds a
  `caffeinate -i` assertion so the Mac does not idle-sleep while a server bills (closing the lid on battery still
  sleeps it). Log: `<state>/<task>/watchdog.log`; `<state>` is `~/.local/state/offload-to-hetzner` unless
  `OFFLOAD_STATE_DIR` says otherwise.
- **Sweeper:** a reboot (an overnight macOS update, say) or a crash kills the watchdog. The sweeper covers that: a
  launchd job that runs `offload sweep` at login and at :00, :15, :30 and :45. It deletes servers more than 2
  minutes past their expiry, orphaned keys and stale local state, so a server whose watchdog died is gone at most
  about 17 minutes after its expiry. A tick missed while the Mac slept runs right after waking. `offload sweeper
  install` writes the job for this Mac (its Python, its `hcloud`, the names of `OFFLOAD_TOKEN_VAR` and
  `OFFLOAD_SECRET_FILES`, never the token), copies the script to `<state>/bin/offload` so plugin updates cannot
  break it, waits for the first run and fails loudly if that run fails. launchd does not see your shell's
  variables, so the token must be in a file of `OFFLOAD_SECRET_FILES` or in an hcloud context. It logs one line per
  run to `<state>/sweeper.log` and shows a macOS notification when it had to delete a server or keeps failing.
  `offload sweeper status|uninstall` checks or removes it; `create` warns when it is missing.
- **Remaining gap:** while the Mac sleeps, is shut down or is offline, nothing deletes anything, so a forgotten
  server bills until the Mac is back (a ccx33 costs about 5 EUR a day). `rm` at the end of every task stays
  mandatory.
- If `create` or its setup fails, server and key are deleted right away.
- `rm <task>...` deletes server and key, checks that neither is listed any more, and only then stops the watchdog;
  if deleting fails, the watchdog stays as the fallback, the other tasks are still removed, and `rm` exits 1. A
  task created with another token (another project) is refused rather than forgotten.
- `ls` shows every offload server in the project with owner, expiry, cost so far, and whether its key is here.
- `clean` removes servers past their expiry, keys without a server, and local state of vanished servers; it skips
  tasks another command is working on, so it is safe at any time. `clean --mine` removes every server of this
  session (owner `cc-<session id>` in Claude Code, else `$OFFLOAD_OWNER`, else your login name), `clean --owner
  <owner>` those of another owner, `clean --all` every offload server; use `--all` only when no other agent is
  working.
- `clean`, `rm` and the safety net touch only servers labelled `managed-by=offload-to-hetzner`, never other
  servers in the same project. Snapshots are never removed by `clean`.
- Confirm at the end that `hcloud server list` and `hcloud ssh-key list` show no `offload-` entries.

## Parallel work

One server per task or shard: `$O create shard-1 ...`, `$O create shard-2 ...`, each with its own push, detached job
and `wait` (in the background), then `get` and `rm` each. Budget and the account limits, not the API, set the
number: every server bills at least one hour.

To separate your failures from existing ones, rerun the failing tests against an untouched checkout of the main
branch on the same server.

## Network and access

- Each server has its own ed25519 key in `<state>/<task>/`, deleted with the server. Only the Mac that ran `create`
  can reach it; other agents on the same Mac can use the same task name.
- The shared firewall `offload-ssh-only` (free, kept between runs) lets in only SSH and ping, so ports published
  by Docker are not reachable from the internet. Outbound traffic is open. `create` checks that an existing
  firewall of that name carries offload's label and exactly these rules, and refuses otherwise.
- The server's SSH host key is made on the Mac and installed by cloud-init, so SSH checks it strictly from the
  first connection on (no trust-on-first-use window). SSH ignores `~/.ssh/config` (`-F /dev/null`) and keeps a
  known_hosts file per server. The host key is part of the server's user-data at Hetzner for the server's lifetime;
  it protects only that throwaway server.

## Settings

| Variable | Default | Meaning |
|---|---|---|
| `OFFLOAD_TOKEN_VAR` | `HCLOUD_TOKEN` | name of the variable that holds the Hetzner token |
| `OFFLOAD_SECRET_FILES` | (none) | colon-separated dotenv files: their values are blocked from uploads, and the token is looked up there |
| `OFFLOAD_STATE_DIR` | `~/.local/state/offload-to-hetzner` | keys, logs, watchdog and sweeper state |
| `OFFLOAD_TYPE` | `cx53` | default server type for `create` |
| `OFFLOAD_KEEP` | `node_modules .venv` | folders `push` keeps on the server |
| `OFFLOAD_GET_MAX_GB` | `20` | size limit for `get` |
| `OFFLOAD_OWNER` | session or login name | owner label, used by `clean --mine` |
