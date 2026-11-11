# Appendix B: Node.js setup for local MCP servers

Many MCP servers ship as npm packages. `npx`, which comes with Node.js, fetches a package when it is not already cached, finds its executable, and runs it. That is how a desktop app, an IDE, or your own agent starts a local server without a custom installer.

Install the current Node 20 LTS or newer: the official installer or a version manager on Windows, Homebrew on macOS, and nvm (or Node's own distro packages) on Linux and WSL. Open a fresh shell and check `node`, `npm`, and `npx`. Node should report v20 or later.

`npx` looks in local `node_modules`, then in its cache, and downloads only if both miss. The first launch is the slow one. A client that spawns the server cannot answer an install prompt, so pass `-y`. The filesystem server is the sanity check: `npx -y @modelcontextprotocol/server-filesystem` plus one allowed directory. It speaks MCP on standard input and output. In client config, the command is `npx` and the arguments are `-y`, the package name, and that directory.

If `npx` is missing, reopen the shell so the path updates. Permission errors usually mean Node was installed as root; switch to a user-level install. If the client says the server died at once, run the same command in a terminal and read the error there. A hang at startup is almost always a missing `-y`. Clear a stale cache with `npm cache clean --force`, or pin a version on the package name, and stay on the current LTS with the same tool you used to install Node.
