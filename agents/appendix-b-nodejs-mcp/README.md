# Appendix B. Node.js for local MCP

Companion notes for **Appendix B** of *AI Agents in Action* (2nd edition,
Micheal Lanham; Manning, 2026).

Most official MCP servers you will actually type in a config are
**Node programs launched with `npx`**. Python runs the agent. Node runs
the **local tool process** on STDIO. Skip this appendix and MCP will look
like a protocol with no sockets: `npx: command not found`, a silent hang
on first launch, or a filesystem server that cannot see the folder you
meant.

This folder is install, verify, cache, one reference server, wiring, and
hygiene. It is the hands-on companion to MCP protocol material: how the
local process actually starts.

## What you are setting up

```
  agent / host (Python SDK, Claude Desktop, IDE)
       |
       |  STDIO  (spawn)
       v
  npx --  @modelcontextprotocol/server-filesystem  (or memory, ...)
       |
       v
  Node process  +  npx download cache
```

The one sentence: **`node` and `npx` must be on the PATH of the process
that spawns the server** — a Node that works in *your* terminal but not
in the debugger, Cursor, or systemd user session is a different machine
as far as MCP is concerned.

## Install Node (Windows, macOS, Linux)

You want a **current LTS** Node, not a random 14.x from 2020. MCP servers
and `npx` package manifests assume a modern runtime. Installers and docs
live at [nodejs.org](https://nodejs.org/).

**Windows.** Use the LTS installer from nodejs.org, or `winget install
OpenJS.NodeJS.LTS`. Close and reopen the terminal (and the editor) so
PATH updates. Git Bash, PowerShell, and cmd can disagree; verify in the
same shell you will use to run samples.

**macOS.** Official LTS installer, Homebrew `brew install node`, or nvm /
fnm if you already manage versions. Apple Silicon vs Intel is the
installer's problem; mixed Homebrew prefixes are yours — `which node`
should be one path.

**Linux.** Distro packages are often **stale**. Prefer NodeSource, nvm,
fnm, or the official binaries over `apt install nodejs` from a two-year
old Ubuntu without checking the version. Snap/flatpak Node can be
invisible to an editor launched from the desktop menu.

Corporate laptops: you may need a blessed installer. The requirement does
not change — **a Node that `npx` can drive**.

## Verify `node` and `npx`

In the same terminal you will use for the book:

```bash
node -v
npx -v
which node
which npx
```

You want a v20 or v22 LTS (or whatever the server's engine field
requires — if `npx` prints an engine warning, believe it). `npx` ships
**with** npm/Node; if `node` works and `npx` does not, PATH or a broken
npm install is the bug, not MCP.

Windows: `where node` and `where npx`. If the editor's integrated
terminal shows different paths than iTerm/Windows Terminal, fix the
editor's environment before debugging Python.

First `npx` of a package **downloads**. That needs network. Offline CI
will fail unless you pre-cache.

## The npx cache

`npx -y some-package` fetches a package into a **cache** (npm's cache
plus npx's installer metadata). The second run is faster. The first run
can look like a hang.

Implications for agents:

- The **user** that spawns MCP (you, a CI user, a service account) owns
  the cache directory. Root vs developer vs debugger can mean three
  caches.
- A poisoned or half-written cache looks like "MCP is flaky." It is
  often a zip that did not finish.
- `-y` skips prompts so a headless agent does not freeze on "Ok to
  proceed?" Never rely on an interactive npx prompt inside a spawn.

You do not need to memorize cache paths. You need to know **clearing it
is a valid fix** (below) and that "works after lunch" was the download
finishing.

## Running the filesystem MCP server

The canonical local smoke test is the official filesystem server: it
exposes a directory as tools (read, list, sometimes write — **treat write
as dangerous**).

Conceptually (package name as published by the MCP project; if a listing
uses a slightly different npm name, follow the listing):

```bash
npx -y @modelcontextprotocol/server-filesystem /path/you/allow
```

Pick a **sandbox directory**, not `$HOME` and not `/`. Chapter samples
that search a repo should be pointed at that repo or a `data/` folder,
not at the whole disk. STDIO MCP still has **your credentials and your
files** — treat the allowed root as a mini threat model.

A healthy run waits on STDIO (it looks idle). That is the server. JSON-RPC
from a client is what makes it print. If it exits immediately, the
argument path is wrong or Node threw — run the same command in a
terminal to see stderr.

Memory graph servers used later in the book are the same class of
process: `npx` plus a package, plus a place they persist JSON. Same PATH,
same cache, same "first run downloads."

## Wiring into a client

**Python Agents SDK.** `MCPServerStdio` (or the current SDK equivalent)
with `command="npx"` and `args` matching what you typed by hand. The
Python process **must** see `npx` on PATH. Debugger launch configs often
have a thin PATH — inherit the login environment or set `command` to an
**absolute** `npx`.

**Claude Desktop / IDE MCP config.** JSON lists `command` and `args`.
Absolute paths to `npx` reduce "works in terminal." Restart the host
after PATH changes; hosts do not always inherit a new shell.

**MCP Inspector** is the right first client when the agent "cannot see
tools." If Inspector lists tools and the agent does not, the bug is the
host config, not Node.

Do not wrap npx in extra shells unless you must. Every wrapper is another
place for quotes and PATH to break.

## Common issues

When a spawn fails, match the symptom to the usual cause before rewriting
the agent.

| Symptom | Likely cause |
|---|---|
| `npx: not found` | Node not installed, or editor PATH |
| Immediate exit, no tools | Bad args; package name; engine too old |
| First call times out | Download, firewall, or waiting on a prompt (add `-y`) |
| Tools exist, reads fail | Allowed directory ≠ the path you think |
| Works in terminal, fails in F5 | Debugger env / cwd / PATH |
| Flaky after a crash | Corrupt npx/npm cache |
| macOS vs Linux CI | Different Node majors; lock the version |

Windows extra: execution policy and "running scripts is disabled" can
block npm's `.cmd` shims. Use an updated Node installer, not a copy of
`npx` from a zip without shims.

## Clearing the cache

When a server "used to work" or extracts throw `ENOENT` / checksum
errors:

```bash
npm cache clean --force
```

Then remove npx's installed package leftovers if your npm docs mention
`~/_npx` or `%LocalAppData%/npm-cache/_npx` (paths vary by npm major).
Re-run the `npx -y ...` command **once** in a terminal until it sits on
STDIO. Then retry the agent.

Clearing the cache is not a personality. It is the fix for a **partial
download**. If it happens every day, the network or disk is the story.

## Updating Node

LTS moves. Once or twice a year:

1. Install the new LTS (nvm `install --lts`, brew, or the Windows
   installer).
2. `node -v` in **editor and CI**, not only in a random window.
3. Re-run one filesystem MCP smoke test. Native addons are rare here;
   engine mismatches are common.
4. Recreate nothing in Python unless a sample says so — Appendix A's
   venv is a different runtime. You can update Node without touching the
   venv.

Do not run production MCP on Node odd-numbered currents unless you like
surprise `engines` failures. LTS is the workshop default.

## You are done when

- [ ] `node -v` and `npx -v` succeed in the **same** shell you use for
  samples (v20/v22 LTS or the server's required engine)
- [ ] Editor / debugger PATH also finds `npx` (or you use an absolute
  path in the MCP config)
- [ ] `npx -y @modelcontextprotocol/server-filesystem /path/you/allow`
  sits on STDIO when run by hand
- [ ] The allowed directory is a sandbox, not `$HOME` or `/`
- [ ] You know to pass `-y` so a headless spawn never waits on a prompt
- [ ] You can clear a corrupt cache with `npm cache clean --force` and
  re-smoke the filesystem server
- [ ] A Python agent (or MCP Inspector) can list tools from that server
