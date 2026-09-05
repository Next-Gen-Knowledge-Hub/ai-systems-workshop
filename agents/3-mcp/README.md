# 3. Actions with MCP

Companion notes for **Chapter 3** of *AI Agents in Action* (2nd edition,
Micheal Lanham; Manning, 2026).

[Chapter 2](../2-llms-prompting-agents/) gave you in-process tools: a
Python function, a schema, a trace span. That is enough for one app.
It is not how an ecosystem works. MCP (Model Context Protocol) is the
**standard socket** so a tool you wrap once can be loaded by a desktop
host, an IDE, and your Agents SDK runtime without rewriting the schema
for each vendor. Skip this chapter and you will copy Slack/GitHub/fs
glue into every agent, debug "the tool" when you are actually debugging
a pipe, and confuse **Claude Desktop clicking a tool** with **an agent
looping until a goal**. Those are different products that happen to
speak the same JSON-RPC.

The Platform track is a different book. If you need "MCP as an org-wide
tool bus, credential isolation, what the protocol does *not* cover,"
that is [platform ch. 6](../../platform/6-tools-and-guardrails/). This
folder stays on **how an agent author speaks MCP**.

## The mental model

Without a protocol, every agent talks to every service in a private
dialect. That is the N×M tax. MCP turns it into **N clients + M
servers** that share one shape.

```
  HOST  (Claude Desktop, Cursor, your Runner, a platform worker)
    |
    |  embeds one MCP client per server
    v
  CLIENT  JSON-RPC 2.0  <--- transport: STDIO  or  SSE/HTTP --->
    |
    v
  SERVER  (your FastMCP process, or `npx` community server)
    |
    |  fronts a real thing
    v
  SERVICE / RESOURCE
    (GitHub API, a folder, a DB, a sequential-thinking store,
     yesterday's in-process `get_research_sources`)
```

The one sentence to remember a year from now: **MCP is USB-C for
capabilities** — tools, resources, and prompts — not a magic runtime
that replaces personas, sampling, guardrails, or a tool registry.

Two consequences fall straight out of that diagram. First, wrapping a
function in `@mcp.tool()` does not make the *agent* safer or smarter;
it makes the function **discoverable** by any host that speaks the
protocol. Second, STDIO and SSE are not "advanced options." They are
**different process topologies**, and picking the wrong one is how
local demos refuse to run in a container, or how a singleton journal
gets cloned per subprocess and "forgets."

Native SDK tools vs MCP vs MCP-on-the-platform:
[TRADEOFFS.md](../../TRADEOFFS.md). Read that table; do not fuse it
into this note.

## MCP fundamentals

Anthropic published MCP as an open JSON-RPC 2.0 convention (2024).
By the time you read this, desktop apps, IDEs, and agent frameworks
all speak some of it. The protocol is the stable part. Each host's
config file and which primitives it actually implements are not.
Always ask: **does this client consume tools only, or also resources
and prompts?**

A recap from [ch. 1](../1-rise-of-ai-agents/): this track introduced
MCP so the five layers had a realistic socket. This chapter is the
deep dive. Keep the five-layer map in your head; MCP can feed more
than layer 2.

### The N×M standardization problem

**Problem** — Before a shared protocol, each model vendor grew a
plugin format, each SaaS grew an SDK, and each agent framework grew
a decorator. A "GitHub tool" written for one stack was a different
object than a "GitHub tool" written for another. Data access was the
same mess: one client for files, one for SQL, one for HTTP, each with
its own auth and error shape. Multi-agent orchestration then invented
a *fourth* way to pass those tools around.

That is N agents × M capabilities: every new host pays M, every new
service pays N.

**Solution** — Wrap the capability once as an **MCP server**. Hosts
implement **one client**. The combinatorial tax becomes addition.

MCP does not, by itself, solve:

- **Who is allowed to call** the tool (policy, IAM, human approval).
- **Where the API key lives** (a secret store).
- **Whether the call is a good idea** (eval, grounding).
- **How you version and discover** tools across teams (a registry).

Those gaps are why [platform ch. 6](../../platform/6-tools-and-guardrails/)
exists. If you pretend the protocol closed them, you will ship a
filesystem server with your cloud keys in the environment and call it
"standardized."

Fragmentation had four faces people mixed up. Keep them separate when
you write an incident doc:

| Face | What broke | What MCP standardizes |
|---|---|---|
| Tool schemas | OpenAI functions vs other vendors' plugins | One list/call shape |
| Data access | Bespoke file/DB/HTTP adapters | Resources (and tools that read) |
| Orchestration glue | Each graph re-binds tools | Hosts share servers |
| Control plane | Ad hoc allow lists in prompts | *Not really* — policy is still yours |

That last row is the trap. A protocol can make control *possible* (you
can inspect what was listed and called). It does not write the policy.

### Clients, servers, and services

USB-C: the cable is the protocol; the laptop socket is the **client**;
the charger is the **server**; wall power is the **service**. Your
agent is not the cable. It is one kind of **host** that holds clients.

Roles, with the words people mix up:

- **Host** — the app the human or the orchestrator started. Claude
  Desktop, Cursor, a Python `Runner` script, a platform worker.
  The host owns UX, permissions prompts, and which servers are on.
- **Client** — the library inside the host that speaks JSON-RPC to
  **one** server. A host with four servers has four client sessions.
- **Server** — a process (or HTTP endpoint) that advertises tools,
  resources, prompts, and implements the calls.
- **Service** — GitHub, a folder, Postgres, Brave, your journal list.
  The server is an adapter in front of it.

**Problem** — "We added MCP" said of a host, a server, and a SaaS in
the same sentence. Nobody knows which layer a timeout belongs to.

**Solution** — Draw the four boxes for *your* failure. A hung `npx`
is a server process. A 401 from GitHub is the service. A host that
never listed tools is a client lifecycle bug. An agent that listed
them and still did not call is a **persona** bug from
[ch. 2](../2-llms-prompting-agents/).

The handshake matters operationally. On connect, the client and
server exchange capabilities. Then the client **lists** tools (and
maybe resources and prompts). The host stuffs those schemas into the
model context — the same token budget you measured in chapter 2.
A chatty server is a prompt tax. Listing is not free.

Servers can also be **composed**: tomorrow someone wraps *your agent*
as a server another host can call. Do not do that until you can run
one server under the Inspector.

### Tools vs resources vs prompts

Every server may expose three primitive types. They are not three
spellings of `def`.

**Tools** — verbs. The agent invokes them to **do** something or to
fetch with arguments: `record_event`, `get_research_sources`,
`create_issue`. They have names, descriptions, JSON-ish argument
schemas, and results. This is what chapter 2's functions were. Most
agent hosts consume tools first; some consume *only* tools.

**Resources** — nouns the host can read (and sometimes subscribe to):
files, configs, a DB view, a URI. Think "open this document" more
than "run this procedure." Good for grounding and for letting a
desktop host show context without a fake tool. Bad as a substitute
for an action that needs arguments and side effects.

**Prompts** — reusable instruction templates the server authors. A
host can offer them as slash-commands or as a starting persona
fragment. They are **not** your agent's whole layer 1. They are
packaged prompts that travel with the capability (e.g. "review this
diff in our style").

```
  list_tools / call_tool          -->  ACT   (layer 2, and others)
  list_resources / read_resource  -->  READ  (often layer 4 fuel)
  list_prompts / get_prompt       -->  SEED  (fragments of layer 1)
```

**Problem** — A client only implements tools, so authors wrap
"read file" and "here is a prompt" as tools to make them visible.

**Solution** — Know you are looking at a **workaround**. It works.
It also pollutes the tool list (token cost, worse selection) and
lies about side effects. If you control the host, implement the
primitive. If you do not, document that your "resource" is a tool
with no mutation, and keep its name honest (`read_*` not `run_*`).

Discovery is part of the product. The descriptions the Inspector
shows are the descriptions the model uses to choose. Docstrings on
`@mcp.tool()` are **prompts**. Treat them with the same care as
chapter 2's function docs: when to call, what args mean, what comes
back, what "empty" means.

Not every server should expose all three. Empty primitives are not
a score.

### Deployment patterns

Where the server process lives is a product choice, not an
implementation leftover.

```
  LOCAL (child process)
    agent  --STDIO-->  server.py / npx ...
    one caller, one lifetime, no port

  LOCAL but out-of-process HTTP
    agent  --SSE-->  localhost:8000
    server already running; several agents can attach

  REMOTE
    agent  --SSE/HTTP-->  cloud host
    auth, latency, shared multi-tenant service

  HYBRID
    secrets and files  -->  local STDIO
    SaaS and search    -->  remote SSE
```

**Local / STDIO.** The host **spawns** the server, talks on stdin/
stdout, and kills it when the session ends. Fast, simple, no open
port. Strictly **one-to-one** with the parent. Great for desktop
and for "this Python file is the server." Terrible as a shared
cache: a new subprocess is a new empty `_journal = []`.

**Local SSE.** You start the server yourself (`mcp run -t sse ...`).
It listens. Agents connect as HTTP clients. Several agents can
share **one** process (and one in-memory store, for better or
worse). You now own a port, a health story, and "is it up?"

**Remote.** Someone else owns the process. You own URL, TLS, auth,
timeouts, and "what if it is slow." This is how community SaaS
servers and internal platform MCP endpoints show up.

**Hybrid.** The grown-up default: keep **high-blast-radius** or
**high-privacy** capabilities on STDIO next to the agent (files,
secrets, prod-adjacent scripts). Put **shared, boring** capabilities
on remote SSE (search, calendar, GitHub). Hybrid is not "use both
transports to look modern." It is a threat-model diagram.

**Problem** — One Compose file, every server on SSE, including the
one that can `rm` the workspace, because "we standardized on HTTP."

**Solution** — Transport follows **trust boundary**. See also
[ch. 8](../8-deploying-agents/) for sandboxing and egress. The
protocol will happily carry a dangerous tool over either wire.

### How MCP feeds the five agent layers

People hear "MCP tools" and park the protocol entirely in layer 2.
That underuses it. A server is a way to **ship a capability** to
whatever layer needs it.

```
  5 Evaluation     MCP: scoring, grounding, rubric servers
  4 Knowledge      MCP: filesystem, Drive, DBs, memory graphs
  3 Reasoning      MCP: sequential thinking, extra working memory
  2 Tools/actions  MCP: GitHub, Slack, calendar, your app APIs
  1 Persona        MCP: prompt templates (fragments, not the whole)
```

**Layer 2 — tools and actions.** The obvious mapping. GitHub issues,
Slack posts, allow-listed HTTP, the research-sources list from
chapter 2. This is also where you must classify **read vs mutate**
the way chapter 1 told you to. An MCP tool that sends mail is not
"just another schema."

**Layer 3 — reasoning and planning.** Some servers do not touch the
world; they give the agent a structured place to think (the
sequential-thinking server is the canonical example). That is
still a tool call in the trace. Deep patterns stay in
[ch. 5](../5-reasoning-and-planning/). MCP is how you **externalize**
the scratchpad so it is inspectable and reusable.

**Layer 4 — knowledge and memory.** Resources and memory-shaped
tools (load journal, graph memory). RAG-the-architecture is
[ch. 6](../6-memory-and-rag/). Here you only need: the protocol can
carry "read this / write that store" the same as it carries
"create issue."

**Layer 5 — evaluation.** A grounding or critic step can be a
server so more than one agent reuses it. The *practice* of eval
is [ch. 7](../7-evaluation-and-feedback/). Do not skip that chapter
because you wrapped a scorer in FastMCP.

**Layer 1 — persona.** Prompt primitives can seed a role. They
should not become the junk drawer of policies chapter 2 warned
about. If a prompt template is versioned with a server, say so in
the host config; do not also paste a divergent copy into
`instructions=`.

When you add a server, **name the layer**. If you cannot, you cannot
write a guardrail or an eval for it later.

## Getting started with MCP servers

A desktop host is the gentlest on-ramp: a config file, a process,
a hammer icon, a permission prompt. It is also a trap if you stop
there, because **desktop hosts are usually assistants**. You will
still use one to learn Inspector, transports, and "does my
decorator match what I think I shipped." Appendix-level Node/`npx`
setup for community servers:
[appendix B](../appendix-b-nodejs-mcp/).

By 2026 the same servers load in multiple IDEs and agent CLIs.
Config keys change; JSON-RPC does not. Learn the primitives, not
one vendor's `mcp.json` folklore.

### Coding a server

The smallest honest server is chapter 2's `get_research_sources`
moved into a FastMCP (or equivalent) process.

What the decorator actually does:

```
  @mcp.tool()
  def get_research_sources() -> list[str]:
      """Provides a list of research sources."""
      ...
```

- **Function name** → tool name the model sees.
- **Signature** → argument schema.
- **Return annotation** → result shape the host may advertise.
- **Docstring** → the only description the agent gets at list
  time. This is a prompt. "Provides a list" is weak; "Return the
  allow-listed sources; call before planning; do not invent
  others" is operational.

FastMCP (and friends) can serve **STDIO by default** when a host
spawns the file, and SSE when you `mcp run -t sse`. Same code,
different entrypoint. That is why the research-tools server can
be used from Claude Desktop *and* from `MCPServerStdio` later
without two implementations.

**Problem** — Tool works in a pytest of the function, fails in the
host.

**Solution** — You tested the Python, not the **schema**. Run the
Inspector. Confirm the advertised name, params, and description.
A missing docstring, a default argument the schema dropped, or a
dict vs list return is a protocol bug, not an LLM bug.

Keep side effects out of import time. `FastMCP("Research Tools")`
at module level is fine. Opening network connections or mutating
global files at import is how STDIO spawn becomes slow and how
SSE reloads surprise you.

Return JSON-serializable types. Print-debugging on stdout **is**
the STDIO transport. Logs on stdout corrupt the RPC stream. Log
to stderr or a file.

### Using the MCP Inspector

The Inspector is a host that is not an LLM. That is the point.
It connects to your server, shows the live tool list, lets you
**call with arbitrary args**, and shows JSON-RPC.

Reach for it when:

- The agent "doesn't see" a tool you registered (schema / name /
  init error).
- A call fails only inside the model loop (reproduce with the
  same args, no persona).
- STDIO vs SSE behave differently (spawn env, cwd, port).
- You need to see initialize errors that a pretty agent runtime
  swallows.

Typical loop: `mcp dev /absolute/path/to/server.py` → open the
printed localhost URL → Tools → run `get_research_sources` →
confirm the three names you expected.

**Problem** — Debugging only through the agent: change prompt,
rerun, shrug.

**Solution** — Binary-search the stack. Inspector succeeds, agent
fails → persona, tool list size, or sampling (chapter 2).
Inspector fails → server, transport, or schema. Host never
connects → command path, `npx`, env, cwd. You cannot skip this
split; the model will happily improvise a tool result in prose.

Use **absolute paths** in spawn configs. Relative paths are how
Desktop and a `Runner` launched from another cwd load different
files that happen to share a name.

### Transport types: STDIO and SSE

Two transports show up everywhere in this chapter. Others exist
in the wild (plain HTTP, stdio-over-SSH hacks). Master these two.

**STDIO**

- Client launches server as a **subprocess**.
- JSON-RPC on the child's stdin/stdout.
- No listen port, low latency, no extra auth on the wire
  (the OS process boundary *is* the auth, plus whatever the
  host prompted the human).
- **One parent, one child.** You cannot attach a second agent
  to that pipe.
- Fits: desktop, CLI, tests, `docker run -it`, "run this file
  as the server."
- Hurts: sharing state across agents, remote hosts, anything
  that must outlive a single run.

**SSE (HTTP + server-sent events)**

- Server already listening; client opens a URL
  (`http://localhost:8000/sse` in the book's-shaped examples).
- Multiple clients can attach.
- Fits: local shared server, containers with a port, remote
  deployment.
- Hurts: more moving parts (bind address, path, health,
  timeouts). Auth is now **your** problem. NAT and proxies
  eat long-lived streams.

```
  STDIO:   host spawns ----pipes----> server   (lifetime coupled)

  SSE:     server runs <----HTTP----  host     (lifetime decoupled)
```

Switching in the Agents SDK is mostly a **constructor swap**
(`MCPServerStdio` vs `MCPServerSse`) plus how you start the
process. The tool names should not change. If they do, you did
not switch transport; you pointed at a different server.

**Problem** — "SSE is production, STDIO is toy," said as a rule.

**Solution** — Production desktop and many CI jobs are correctly
STDIO. Production *shared* tool farms are correctly SSE/HTTP.
The [TRADEOFFS.md](../../TRADEOFFS.md) transport table is the
one-screen version. Measure process startup too: STDIO pays spawn
cost every run; SSE pays an always-on server.

### Desktop vs agent hosts

Claude Desktop (and ChatGPT / IDE MCP panels) will load your
server. That is useful and **not the same execution model** as
the Agents SDK.

Permission prompts are a **security UX**, not the definition of
assistant vs agent. Plenty of coding agents ask before `rm`.
Plenty of assistants auto-run a search. Do not use the modal as
your taxonomy.

The difference that matters is the **loop** from
[ch. 1](../1-rise-of-ai-agents/):

```
  ASSISTANT HOST (typical desktop chat)
    user turn -> maybe a few tool calls -> answer -> STOP
    human is the scheduler for the next goal

  AGENT HOST (Runner, your graph)
    goal -> tool -> observe -> plan -> tool -> ...
    until termination / budget / guardrail
    human is not in the loop each hop
```

**Problem** — A server "works in Desktop" so it is declared
agent-ready. Under a Runner it loops, double-writes, or never
stops.

**Solution** — Test with an **autonomous** host. Idempotent tools,
clear "already recorded" results, and personas that say when to
stop matter more once nobody clicks each call. Desktop success
means: schema loaded, one-shot call returned. Agent success
means: SPAL over many calls still makes sense.

Desktop users see a toast. Agents see a string observation and may
retry until the budget dies. Return structured errors.

## Using MCP with agents

Now the host is the OpenAI Agents SDK (or the same pattern
elsewhere): `mcp_servers=[...]` on the `Agent`, Runner as in
[ch. 2](../2-llms-prompting-agents/). The model does not "speak
MCP." The runtime lists tools, stuffs schemas into the completion,
executes calls, and feeds observations back. MCP is behind the
same tool span you already learned to read.

Actions, in this track's vocabulary, are whatever the agent does
to finish a goal: tools, resource reads, even a handoff. MCP is
how many of those actions get **hosted**.

### Local MCP over STDIO

Pattern: the agent process **owns** the server process.

```
  SCRIPT = Path(__file__).with_name("server.py").resolve()

  async with MCPServerStdio(
      name="Research Tools",
      params=MCPServerStdioParams(
          command="mcp",
          args=["run", str(SCRIPT)],
      ),
  ) as research_server:
      agent = Agent(
          name="Assistant",
          instructions="Use the research tools ...",
          mcp_servers=[research_server],
      )
      result = await Runner.run(agent, "Get the available ...")
```

Why this shape:

- **`async with`** ties session lifetime to the child. No orphan
  `mcp` processes if you got the wait right.
- **Absolute script path** so cwd does not pick another file.
- **Instructions still matter.** The server can list tools; the
  persona still has to say to use them (chapter 2's lesson does
  not expire).
- **Async runner.** STDIO and RPC sit badly in a naive sync
  story once anything waits.

**Problem** — You edit the server, rerun the agent, and see old
tool descriptions.

**Solution** — You are looking at a spawned process that loaded
bytecode or a different path. Confirm `SCRIPT`, kill strays, use
Inspector against the same command the SDK runs. Caching and
"helpfully" reused sessions will gaslight you.

In-memory globals on the server **reset** when the child dies.
For a research-sources list that is a feature (pure function).
For a journal, STDIO means **the diary dies with the run** unless
you persist in the service (a file, a DB) instead of a module
list.

### Local MCP over SSE

When in-process spawn is the wrong topology — slow startup, need
several agents, want to keep a journal alive — run the server as
an HTTP process and attach.

```
  # terminal 1
  mcp run -t sse 01_claude_mcp_server.py

  # agent process
  async with MCPServerSse(
      name="SSE Python Server",
      params={"url": "http://localhost:8000/sse"},
  ) as research_server:
      ...
```

Same agent instructions, same tool names, different **how we
connect**. That is the pedagogical point of flipping STDIO → SSE
without changing the persona.

Consequences you must actually handle:

- **Start order.** The agent is not your process supervisor.
  If the URL is down, you get a client error, not a spawn.
- **Shared state.** Two agents, one SSE server, one `_journal`:
  they share memory. That is either a feature (coordination) or
  a bug (tests contaminate each other).
- **Host binding.** `localhost` from inside Docker is not your
  laptop. Compose networks and published ports are
  [ch. 8](../8-deploying-agents/) territory; learn the words now.
- **Fan-out.** Multiple clients are why you picked SSE. They
  also mean you need to think about concurrency in the server
  (a global list without a lock is a demo).

**Problem** — FastMCP defaults on STDIO when spawned, SSE when
run with `-t sse`, and someone hardcodes the port in three
files.

**Solution** — One documented URL per environment. Treat it like
a DSN. The constructor swap is easy; the **ops** around SSE is
the real cost, which is why STDIO remains correct for many
agents.

### Standard and community servers

You should not wrap GitHub by hand on week one. Reference and
community servers exist: filesystem, sequential thinking, Drive,
calendar, search, and a long tail installable with `npx` (Node)
or equivalent. [Appendix B](../appendix-b-nodejs-mcp/) is the
Node on-ramp this track expects.

How to consume them like an adult:

- **Read the tool list in Inspector** before you hand them to an
  agent. Community servers are large. Fifteen filesystem verbs
  will swamp a small persona.
- **Pin versions.** `npx -y @scope/pkg` without a version is a
  supply-chain and a behavior lottery.
- **Scope filesystem roots.** A server that can read `$HOME` is
  not a "standard tool"; it is a data-exfiltration primitive.
  Pass an explicit directory.
- **Name the layer.** Sequential thinking is layer 3. Filesystem
  is usually 2 + 4. Calendar mutate is layer 2 with irreversible
  consequences.
- **Credentials.** Drive and GitHub servers need tokens. Those
  tokens in a desktop config file are a different threat than
  tokens in a platform secret store —
  [platform ch. 6](../../platform/6-tools-and-guardrails/).

**Problem** — Adding five `npx` servers to look capable. The
model spends the window reading schemas and then calls nothing
useful.

**Solution** — Minimum servers for the goal. Chapter 2's "short
tool list" rule survives the protocol change. Capability is not
measured in MCP rows in a JSON config.

When a community server is almost right, configure it; do not fork
it for one flag. Forks miss protocol updates.

## Building servers for agents

The last move in the chapter is the one you will repeat at work:
**stop registering in-process functions on the Agent; host them.**
The time-tracker / journal example is the teaching story: a loop
records events with `record_event`, then a summary uses
`load_journal`. First both functions live in the agent file.
Then they move.

Why bother if the Python is the same:

- **Reuse.** A second agent (or Desktop, or an IDE) can load the
  journal tools without importing your app.
- **Separation.** Persona and sampling stay in the host; I/O
  stays in the server. You can eval them apart.
- **Deployment choice.** The same server file can be STDIO today
  and SSE tomorrow.
- **Honesty about state.** Once the journal is a service, you
  have to say where it lives. A hidden global in the agent
  module pretended not to be a database.

### Converting tools to a server

Mechanically: new module, `FastMCP("Time Travel Tracker")`,
decorate the same functions, keep docstrings as prompts, keep
types JSON-friendly.

```
  BEFORE                         AFTER
  agent.py                       tracker_mcp.py
    record_event  -------+         @mcp.tool record_event
    load_journal  -------+         @mcp.tool load_journal
    Agent(tools=[...])             _journal = []   # now a question
         |                              |
         v                              v
    Runner                         agent.py
                                   Agent(mcp_servers=[...])
```

The agent constructor **drops** `tools=[functions]` and **gains**
a server session. Discovery replaces import. If a tool
"disappears," it is a list/handshake failure, not a missed
import.

**Problem** — `_journal = []` at module scope, shipped as if it
were durable.

**Solution** — Name the store. In-memory is allowed for a lab.
Then pick transport knowing the lifetime (STDIO wipes it;
long-lived SSE keeps it until restart). For anything real, the
server fronts a file or DB, and you have a concurrency story.
MCP does not give you persistence. It gives you a **door**.

Docstrings often need a rewrite at the boundary. In-process,
"add a new travel event" was enough because the developer sat
next to the code. Across a protocol, say: arguments, idempotency
("duplicate entry names append or reject?"), and that
`load_journal` should be called **before** recording if the
persona must not clobber context. Those sentences are layer-1
adjacent, but they travel with the tool for every host. Keep
host-specific policy in the agent's `instructions`.

Converting is also when you delete dead tools. A function that
existed "for the demo print" should not become a networked verb.

### Consuming local vs remote

After the split, the host chooses a transport. Conceptually:

```
  LOCAL STDIO     spawn `mcp run tracker_mcp.py`
                  one agent, fresh memory unless you persist

  LOCAL SSE       `mcp run -t sse tracker_mcp.py` then URL
                  shared memory among agents on that host

  REMOTE SSE      URL + auth to someone else's tracker
                  you consume; they operate
```

The SDK `with MCPServerStdio(...)` / `MCPServerSse(...)` block
plus `mcp_servers=[session]` is the whole client story on
purpose. You should feel slightly bored by the swap. The
**unboring** parts are env vars, secrets, network, and state.

When consuming **remote**:

- Timeouts and retries are host policy. A hung remote tool is
  an agent-loop failure mode ([ch. 9](../9-agentic-loop/) will
  make that painful).
- Auth headers do not belong in tool arguments the model can
  see. They belong in the client session config.
- Schema drift: their server updated; your persona still
  describes old args. Inspector against *prod* (read-only
  tools first) is cheaper than guessing from a README.

When consuming **local**:

- You still need a threat model. Local filesystem MCP is full
  access to whatever root you passed.
- Dev/prod parity: the STDIO command in tests should be the
  command you think Desktop uses.

**Problem** — Agent instructions copy the entire API of the
server, then the server changes.

**Solution** — Instructions say **intent** ("always load the
journal first; record new events; summarize from the tool, do
not invent"). Names of tools should match, but do not paste
argument lists that the schema already provides. Duplicated
schemas drift — the same rule as `output_type` in chapter 2.

You can mix: native function tools *and* MCP servers on one
agent. Do that when a tool is truly private to the process
(tiny, no reuse). If you find a second agent needing it,
convert. That is the whole migration heuristic.

## See also

Same protocol, different job — do not merge the chapters.

- **[platform ch. 6 — tools and guardrails](../../platform/6-tools-and-guardrails/)**
  MCP on a **platform**: registry, versions, adapters, credential
  isolation, sync vs async execution, resource limits, circuit
  breakers. Explicitly: **what MCP does not cover** (policy,
  secrets, org discovery). This folder taught you to wrap and
  consume a server. That folder teaches you why five teams
  wrapping the same GitHub server is still sprawl.
- Native vs MCP vs platform MCP:
  [TRADEOFFS.md](../../TRADEOFFS.md).
- Sequential thinking as a layer-3 server:
  [ch. 5](../5-reasoning-and-planning/).
- Node/`npx` for community servers:
  [appendix B](../appendix-b-nodejs-mcp/).

## Check yourself

1. Draw N=3 hosts and M=4 services with bespoke plugins, then
   again with MCP. Where does the remaining work live (you still
   pay N+M)? Give a real internal API at your job that would
   still need an adapter behind the server.
2. A timeout fires. Using host / client / server / service, pick
   the box you inspect first for (a) `npx` never started, (b)
   GitHub 401, (c) tools never listed in the trace, (d) tools
   listed but the model invented a URL. Why is (d) not an MCP
   bug?
3. Your IDE client "doesn't support resources," so you expose
   `read_handbook` as a tool. What did you gain, what did you
   pay in the tool list, and when would you unwind the
   workaround?
4. A journal server uses `_journal = []`. Predict the contents
   after three events under STDIO (one run), after a second
   `Runner` process, and under one long-lived SSE with two
   agents. Which topology matches a demo, which matches a shared
   store, and what still is not durability?
5. You put a filesystem MCP on SSE in Kubernetes "for
   consistency," with the pod able to read a secrets volume.
   Which deployment-pattern rule did you break, and what
   transport would you use instead for that server?
6. Map each to a layer: GitHub `create_issue`, sequential
   thinking, a Drive `read_resource`, a rubric scorer, a prompt
   template named `incident_commander`. Which one is most often
   parked in the wrong layer, and what fails when you do?
7. Inspector can call `get_research_sources`; the agent never
   does. Name two chapter 2 causes and one MCP-specific cause
   (handshake, path, transport). What is the cheapest experiment
   that distinguishes them?
8. Why is "it works in Claude Desktop" not evidence that the
   same server is safe under `Runner.run` for a 20-step goal?
   Mention the loop, not the permission modal.
9. You convert in-process tools to MCP and the agent still
   imports the functions "as a backup." What have you failed to
   separate, and what bug class (schema drift, double write)
   becomes likely?
10. A platform teammate says "we don't need this chapter, we'll
    just use the Tool Service." Which part of MCP do you still
    have to understand as an agent author, and which part
    (credentials, registry, what MCP does not cover) is actually
    their folder?

Continue to [Multi-agent systems](../4-multi-agent-systems/).
