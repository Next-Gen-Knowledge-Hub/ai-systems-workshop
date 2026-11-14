### **Appendix B Summary: Node.js Setup for Local MCP Servers**

Appendix B of *AI Agents in Action, Second Edition* prepares a **Node.js LTS toolchain** so MCP servers published on npm can be launched with `npx` from agents, IDEs, or desktop clients without custom install scripts.

---

#### **1. Installing Node.js**

* **LTS baseline**: MCP examples assume **Node 20 LTS or newer**, which bundles `npm` and `npx` together.
* **Windows options**: Use the official installer from nodejs.org or **nvm-windows** when multiple Node versions must coexist.
* **macOS path**: Homebrew (`brew install node`) or the macOS installer from nodejs.org are the usual routes.
* **Linux and WSL**: **nvm** installs user-local Node without sudo; distribution packages via NodeSource are an alternative when nvm is undesirable.

---

#### **2. Verifying Node and npx**

* **Version checks**: Fresh shells should report versions for `node`, `npm`, and `npx`, with Node at **v20+**; “command not found” means PATH or nvm activation was skipped.
* **npx resolution**: `npx` prefers local `node_modules`, then the global npx cache under the user home, downloading from npm only on a cache miss—so the **first run is slower** than later ones.

---

#### **3. Running an MCP Server with npx**

* **Command shape**: Typical launches follow `npx -y <package> [args...]`, where **`-y` suppresses the install prompt** subprocess clients cannot answer.
* **Filesystem example**: `npx -y @modelcontextprotocol/server-filesystem` plus an allowed directory validates the server in a terminal; it speaks MCP over stdio until stopped with Ctrl-C.
* **Client configuration**: MCP JSON configs set `"command": "npx"` and pass `-y`, the package name, and server-specific arguments as the `args` array—the same pattern works for other npm-hosted MCP servers.

---

#### **4. Troubleshooting and Keeping Node Healthy**

* **Common failures**: Missing `npx` after install needs a new shell; **EACCES** often traces to sudo-installed Node; immediate client exit deserves a direct terminal run to surface stderr; hangs frequently mean `-y` was omitted.
* **Cache hygiene**: `npm cache clean --force` or pinning `@version` on the package name forces a fresh download when a stale cached server misbehaves.
* **Staying current**: Re-run installer, `brew upgrade node`, or `nvm install --lts` periodically, then repeat the three version checks so agents keep using supported Node releases.
