# offload-to-hetzner

**Offload heavy builds, Docker, test suites and parallel AI-agent work from your Mac to short-lived Hetzner Cloud
servers.** A Claude Code plugin and Agent Skill, and the Hetzner counterpart to Vercel's
[`offload-to-sandbox`](https://github.com/vercel/vercel-plugin/tree/main/skills/offload-to-sandbox) skill for
Vercel Sandbox.

One laptop can't keep up with ten agents that all want to run Docker and full test suites at the same time. This
skill gives each task its own throwaway Linux server: create it, push the code, run the job detached, fetch the
results, delete it. A Hetzner `cx53` with 16 cores and 32 GB of RAM costs about **0.05 EUR per hour**.

```text
offload create parser-tests --type cx53 --setup docker
offload push   parser-tests .
offload exec   parser-tests --detach suite --cwd /root/work/myrepo -- 'mkdir -p /root/results && make test > /root/results/test.txt'
offload wait   parser-tests suite
offload get    parser-tests /root/results ./offload-results
offload rm     parser-tests
```

## Inspired by Vercel's offload-to-sandbox

Vercel's [`offload-to-sandbox`](https://vercel.com/docs/agent-resources/skills) skill showed the pattern: agents
edit and orchestrate locally, and everything heavy goes to a remote machine of its own. This project brings the
same workflow to [Hetzner Cloud](https://www.hetzner.com/cloud/). It is an independent project, not affiliated
with Vercel or Hetzner, and shares no code with Vercel's skill.

The trade-off, with prices from the providers' pricing pages and API (Vercel Pro, `iad1`, September 2026;
Hetzner API, November 2026, without VAT):

| | Vercel Sandbox (Pro) | Hetzner `ccx33` | Hetzner `cx53` |
|---|---|---|---|
| Size | 8 vCPU, 16 GB (Pro maximum) | 8 dedicated vCPU, 32 GB | 16 shared vCPU, 32 GB |
| Per hour, CPU fully busy | about 1.36 USD | 0.19 EUR in Helsinki (0.22 in Germany) | 0.05 EUR |
| Per hour, agent mostly waiting (10 % CPU) | about 0.44 USD | 0.19 EUR in Helsinki | 0.05 EUR |
| Billing | active CPU per ms + memory | per started hour | per started hour |
| Start | seconds | about 1 minute (+2 min for Docker, or a snapshot) | same |
| Expiry | built in | **none at Hetzner**: this skill adds watchdog + sweeper | same |

Pick Vercel when start time and per-millisecond billing matter, or when you need thousands of sandboxes at once.
Pick Hetzner for builds and test suites that keep CPUs busy: several times cheaper, with twice the RAM per core.

## When you don't need this

A throwaway server per task pays off when heavy work comes in bursts and several agents run at once. Often
something simpler is enough:

- **One project, steady load:** keep one server. A Hetzner auction server (dedicated hardware for a flat monthly
  price) or a cloud server that stays on needs no creating and no cleanup; give your agent SSH access and you are
  done.
- **The whole session on a server:** Claude Code can run a session on a remote machine over SSH, from the desktop
  app or with `claude` in a terminal on the server. Then nothing heavy runs on the Mac in the first place. The
  server runs all month, though, and whatever exists only on your Mac (its browser, local files, the keychain) is
  not there.
- **A machine now and then, by hand:** `hcloud server create` and `ssh` are all you need. This tool is a thin layer
  on top of the `hcloud` CLI.

Use this tool when your agents should stay on your Mac and rent machines only for the heavy parts, several at a
time, without anyone having to remember to delete them.

## What it takes care of

Hetzner has no automatic expiry, so a forgotten server bills until someone deletes it. Most of this skill is
about making that hard to get wrong:

- **Three layers of cleanup.** Every server has a lifetime (`--ttl`, default 4 h). A watchdog process on your Mac
  deletes server and key at expiry, also right after the Mac wakes up, and keeps the Mac from idle-sleeping
  meanwhile. A launchd job (the sweeper) runs every 15 minutes and after login, and deletes what a reboot or crash
  left behind, with a macOS notification. `rm` checks that server and key are really gone before it says so.
- **Only its own servers.** Everything it deletes carries the label `managed-by=offload-to-hetzner`; your other
  servers in the same project are never touched.
- **Secrets stay home.** Every upload and every remote command line is scanned for the values of your secret
  environment variables, your project's `.env` files and common token formats, in contents, file names and link
  targets. Uploads are packed from exactly the bytes that were scanned. Nothing is sent when the scan finds
  something.
- **Results are untrusted.** `get` accepts only plain files below the folder you asked for, never merges into or
  overwrites existing files unless you pass `--merge`, never writes into your home folder or a `Library` folder,
  never writes through symlinks, refuses `.git` and git control files, drops execute bits and has a size limit.
- **Locked-down servers.** Each server gets its own SSH key and a host key made on your Mac (no trust on first
  use), and a firewall that only lets SSH and ping in. Servers get an IPv4 address only, so each uses one of your
  account's primary IPs instead of two.
- **Parallel agents.** Per-task locks, owner labels per Claude Code session, and `clean --mine` for a session's
  own servers.
- **Base snapshots.** Install a toolchain once and start servers from it in about 45 seconds.

## Install

Requirements: macOS, Python 3.11+, the [`hcloud` CLI](https://github.com/hetznercloud/cli), and a Hetzner Cloud
API token with read & write access (Hetzner Console, your project, Security, API tokens).

```bash
brew install hcloud python
```

**As a Claude Code plugin:**

```text
/plugin marketplace add pipamann/offload-to-hetzner
/plugin install offload-to-hetzner@offload-to-hetzner
```

**As an Agent Skill** for Claude Code, Codex, Cursor and other agents that read `SKILL.md` files:

```bash
npx skills add pipamann/offload-to-hetzner
```

The wrapper is a plain command-line tool, so you can also clone the repo and run
`skills/offload-to-hetzner/offload` yourself.

## Set up the token and the sweeper

The token is looked up in this order: the variable `HCLOUD_TOKEN` (or the one named in `OFFLOAD_TOKEN_VAR`), the
same name in the dotenv files listed in `OFFLOAD_SECRET_FILES`, then hcloud's active context. The simplest setup
is an hcloud context:

```bash
hcloud context create offload    # paste the read & write token
```

Then install the sweeper once (ask your agent to do it, or run it from the skill folder):

```bash
offload sweeper install          # launchd job: cleanup at login and every 15 minutes
```

launchd does not see your shell's variables, so the sweeper reads the token from an hcloud context or from a file
in `OFFLOAD_SECRET_FILES`. `install` waits for the first run and fails loudly if the job cannot work.

Tip: create a separate Hetzner project just for offload servers. A token is limited to its project, so the token
on your Mac then cannot touch your production servers.

## Usage

Your agent reads [`SKILL.md`](skills/offload-to-hetzner/SKILL.md) and proposes a server with type, time and cost
before creating it. `offload --help` lists every command:

| Command | What it does |
|---|---|
| `types` | live prices of all server types |
| `create <task> [--type] [--ttl] [--setup docker\|<file>] [--from <base>]` | new server with key, labels, expiry, watchdog |
| `push <task> [<dir>] [<remote-dir>]` | sends git-tracked and untracked files; keeps `node_modules`/`.venv` on the server |
| `put` / `get` | copy files or folders up and down |
| `exec <task> [--detach <job>] [--cwd <dir>] -- '<cmd>'` | run now, or as a background job that survives dropped connections |
| `wait` / `logs` | wait for a job and get its exit status; show or follow its log |
| `ls` / `rm` / `clean` | list with cost so far; delete and verify; remove expired or orphaned resources |
| `sweeper install\|status\|uninstall` | the launchd safety net |
| `base create\|ls\|rm` | snapshots to start servers from |

Account limits: Hetzner limits shared vCPUs, dedicated vCPUs and primary IPs per account, across all projects.
New accounts start low (for example 8 dedicated vCPUs). Requests in the Hetzner Console (Limits, "Request limit
increase") can be approved within seconds.

## Limitations

- macOS only for now: the watchdog and sweeper use `caffeinate`, launchd and macOS notifications. Linux support
  is welcome as a pull request.
- While your Mac sleeps, is off or offline, nothing deletes anything. A forgotten server bills until the Mac is
  back; end every task with `offload rm`.
- The secret scan cannot recognise secrets it does not know and that have no known format, or customer data in
  ordinary files.
- Hetzner bills every started hour, even for a five-minute server.
- Hetzner has restricted the creation of new cloud servers since June 2026 for new customers and some existing
  ones, citing hardware shortages. If `create` is refused although your limits allow it, that is why.
- No GPUs: Hetzner Cloud has none (Hetzner's GPU servers are dedicated machines with a setup fee per order), and
  neither has Vercel Sandbox. For GPU work, use a GPU cloud such as Modal.

## License

MIT. No warranty: this tool creates resources that cost money in your Hetzner account; check `offload ls` or the
Hetzner Console when in doubt.
