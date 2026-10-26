# Appendix B. Node.js for local MCP

Companion notes for **Appendix B** of *AI Agents in Action* (2nd edition, Micheal Lanham; Manning, 2026).

Most official MCP servers you type into a config are Node programs launched with `npx`. Python runs the agent. Node runs the local tool process on STDIO. Without Node on the PATH of the process that spawns the server, you get `npx: command not found`, a silent hang on first launch, or a filesystem server pointed at the wrong folder. A Node that works in your terminal and fails in the debugger, the editor, or a systemd user session is a different machine as far as MCP is concerned.

## 1. What you are setting up

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

* **Who spawns whom**: The host (a Python SDK, Claude Desktop, or an IDE) spawns `npx`. The Node process speaks JSON-RPC on STDIO.
* **PATH is per process**: `node` and `npx` have to be on the PATH of the spawning process. A terminal you opened by hand can see a Node the debugger cannot.

## 2. Install Node

You want a **current LTS** Node, not a random 14.x from 2020. MCP servers and `npx` package manifests assume a modern runtime. Installers and docs live at [nodejs.org](https://nodejs.org/).

* **Windows**: Use the LTS installer from nodejs.org, or `winget install OpenJS.NodeJS.LTS`. Close and reopen the terminal and the editor so PATH updates. Git Bash, PowerShell, and cmd can disagree. Verify in the same shell you will use to run samples.
* **macOS**: Official LTS installer, Homebrew (`brew install node`), or nvm / fnm if you already manage versions. Apple Silicon versus Intel is the installer's problem. Mixed Homebrew prefixes are yours. `which node` should be one path.
* **Linux**: Distro packages are often stale. Prefer NodeSource, nvm, fnm, or the official binaries over `apt install nodejs` from a two-year-old Ubuntu without checking the version. A Snap or flatpak Node can be invisible to an editor launched from the desktop menu.
* **Corporate laptops**: You may need a blessed installer. The requirement is the same: a Node that `npx` can drive.

## 3. Verify `node` and `npx`

In the same terminal you will use for the samples:

```bash
node -v
npx -v
which node
which npx
```

* **Version**: You want a v20 or v22 LTS, or whatever the server's `engines` field requires. If `npx` prints an engine warning, believe the warning.
* **`npx` ships with Node**: If `node` works and `npx` does not, PATH or a broken npm install is the bug.
* **Windows**: Use `where node` and `where npx`. If the editor's integrated terminal shows different paths than iTerm or Windows Terminal, fix the editor's environment before you debug the Python agent.
* **First download**: The first `npx` of a package downloads and needs a network. Offline CI fails unless you pre-cache that package.

## 4. Understand the npx cache

`npx -y some-package` fetches a package into a cache (npm's cache plus npx's installer metadata). The second run is faster. The first run can look like a hang.

* **Cache owner**: The user that spawns MCP (you, a CI user, a service account) owns the cache directory. Root, your user, and the debugger can mean three caches.
* **Half-written cache**: A poisoned or half-written cache looks like a flaky server. It is often a zip that did not finish.
* **`-y`**: The flag skips the prompt so a headless agent does not freeze on "Ok to proceed?" An interactive npx prompt inside a spawn waits forever.
* **Two facts worth keeping**: Clearing the cache is a valid fix. "Works after lunch" was the download finishing.

## 5. Run the filesystem server

The local smoke test is the official filesystem server. It exposes a directory as tools: read, list, and sometimes write. Treat write as dangerous.

The package name below is the one published by the MCP project. If a listing uses a slightly different npm name, follow the listing.

```bash
npx -y @modelcontextprotocol/server-filesystem /path/you/allow
```

* **Sandbox directory**: Point the server at a sandbox directory, not `$HOME` and not `/`. A sample that searches a repo should use that repo or a `data/` folder, not the whole disk. A STDIO server still has your credentials and your files. Treat the allowed root as a small threat model.
* **Healthy idle**: A healthy run waits on STDIO and looks idle. That is the server. JSON-RPC from a client is what makes it print. An immediate exit means the path argument is wrong or Node threw. Run the same command in a terminal and read stderr.
* **Same shape, other packages**: A memory-graph server is the same class of process: `npx`, a package, and a place it persists JSON. Same PATH, same cache, same first-run download.

## 6. Wire the server into a client

* **Python Agents SDK**: Use `MCPServerStdio` (or the current SDK equivalent) with `command="npx"` and `args` that match the command you already ran by hand. The Python process has to see `npx` on PATH. Debugger launch configs often carry a thin PATH. Inherit the login environment, or set `command` to an absolute `npx`.
* **Claude Desktop or an IDE**: The JSON config lists `command` and `args`. An absolute path to `npx` keeps the host aligned with the terminal. Restart the host after a PATH change. Hosts do not always pick up a new shell.
* **MCP Inspector first**: When the agent cannot see tools, Inspector is the right first client. If Inspector lists the tools and the agent does not, the bug is the host config.
* **Extra shells**: Skip a wrapper shell around `npx` unless you have to have one. Every wrapper is another place for quotes and PATH to break.

## 7. Match the symptom

| Symptom | Likely cause |
|---|---|
| `npx: not found` | Node is not installed, or the editor PATH is missing it |
| Immediate exit, no tools | Bad args, wrong package name, or an engine that is too old |
| First call times out | Download, firewall, or a prompt waiting for input (add `-y`) |
| Tools exist, reads fail | The allowed directory is not the path you think |
| Works in the terminal, fails under F5 | Debugger env, working directory, or PATH |
| Flaky after a crash | Corrupt npx or npm cache |
| macOS works, Linux CI fails | Different Node majors; lock the version |

* **Windows script policy**: An execution policy of "running scripts is disabled" can block npm's `.cmd` shims. Use an updated Node installer. Copying `npx` out of a zip that has no shims leaves the same block in place.

## 8. Clear a corrupt cache

Use this when a server used to work, or when extracts throw `ENOENT` or a checksum error.

```bash
npm cache clean --force
```

* **Leftover installs**: Remove npx's installed package leftovers when your npm docs mention `~/_npx` or `%LocalAppData%/npm-cache/_npx`. Those paths vary by npm major.
* **Re-smoke by hand**: Run the `npx -y ...` command once in a terminal until it sits on STDIO. Then retry the agent.
* **What the clean fixes**: A partial download. A cache you have to clear every day means the network or the disk is the story.

## 9. Update Node

LTS moves. Once or twice a year:

1. **Install the new LTS**: `nvm install --lts`, Homebrew, or the Windows installer.
2. **Check the environments that spawn servers**: Run `node -v` in the editor and in CI, not only in a random window.
3. **Re-smoke**: Run one filesystem MCP smoke test. Native addons are rare on these servers. An `engines` mismatch is common.
4. **Leave the Python environment alone**: Recreate the Python virtual environment only when a sample says to. Node and that virtual environment are different runtimes. You can update Node without touching it.

* **Stay on LTS**: An odd-numbered current release is a common source of surprise `engines` failures. LTS is the default for this setup.

## You are done when

- [ ] `node -v` and `npx -v` succeed in the same shell you use for samples (v20 or v22 LTS, or the server's required engine)
- [ ] The editor and debugger PATH also find `npx`, or the MCP config uses an absolute path
- [ ] `npx -y @modelcontextprotocol/server-filesystem /path/you/allow` sits on STDIO when you run it by hand
- [ ] The allowed directory is a sandbox, not `$HOME` or `/`
- [ ] Headless spawns pass `-y`, so they never wait on a prompt
- [ ] You can clear a corrupt cache with `npm cache clean --force` and re-smoke the filesystem server
- [ ] A Python agent, or MCP Inspector, can list tools from that server

## Check yourself

1. `node -v` works in your terminal, and the agent reports `npx: not found` under the debugger. What differs between those two processes, and what do you put in `command` so the debugger stops depending on its own PATH?
2. The first filesystem spawn times out. A second run in the same terminal sits idle on STDIO. What was the first run doing, and which flag keeps a headless spawn from waiting on "Ok to proceed?"
3. You pointed `@modelcontextprotocol/server-filesystem` at `$HOME` and the server started. What can that process reach, and which directory should a repo sample use instead?
4. MCP Inspector lists the filesystem tools and the Python agent does not. Which side is broken, the Node install or the host config?
5. Extracts throw `ENOENT` after a crash mid-download. Which command clears npm's cache, which leftover directory do the npm docs name, and what do you run once by hand before the agent tries again?
6. After a Node upgrade, a server logs an `engines` warning. Why does recreating the Python virtual environment leave that warning in place?
