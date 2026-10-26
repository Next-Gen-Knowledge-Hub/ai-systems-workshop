# 3. Actions with MCP

Companion notes for **Chapter 3** of *AI Agents in Action* (2nd edition, Micheal Lanham; Manning, 2026).

This chapter is the socket. It establishes how a host, a client, a server, and a service share one JSON-RPC shape, so a capability wrapped once can be loaded by a desktop app, an IDE, and an agent runtime. Afterward you can name the transport, the primitive (tool, resource, or prompt), and the agent layer that server feeds. A dead process, a rejected credential, and a model that never called the tool stop looking like the same bug.

## 1. One socket instead of a private dialect

Without a shared protocol, every agent talks to every service in a private dialect. That is an N×M tax: every new host pays for every capability, and every new capability pays for every host. **MCP** (Model Context Protocol) is an open JSON-RPC 2.0 convention, published by Anthropic in 2024, that turns the tax into N clients plus M servers.

```
  HOST   (desktop app, IDE, your Runner, a platform worker)
    |
    |  one MCP client session per server
    v
  CLIENT   JSON-RPC 2.0
    |      transport: STDIO  or  SSE/HTTP
    v
  SERVER   (your FastMCP process, or an npx community server)
    |
    v
  SERVICE  (GitHub, a folder, a database, a scratchpad,
            yesterday's in-process get_research_sources)
```

* **What the socket carries**: Tools, resources, and prompts. People call it USB-C for those capabilities: one cable, many devices. Discovery, the call shape, and the response shape are the standardized part.
* **What the socket does not decide**: Who may call, where the API key lives, whether the call was a good idea, or how a company versions and finds a tool. Those stay policy, secrets, evaluation, and registry work. A filesystem server with cloud keys in the environment is not "standardized." It is a wrapped secret.
* **Discoverable is not safer**: `@mcp.tool()` makes a function visible to any host that speaks the protocol. It does not make the agent wiser, and it does not add a guardrail.
* **Two wires, two topologies**: STDIO and SSE are process layouts. The wrong one is how a local demo refuses to run in a container, or how a singleton journal is cloned per subprocess and forgets.

The protocol is the stable part. Each host's config file, and which primitives that host actually implements, are not. Desktop apps, IDEs, and agent CLIs invent their own config keys. The list and call messages stay JSON-RPC 2.0. Always ask whether this client consumes tools only, or also resources and prompts.

## 2. Four faces of the old mess

Fragmentation had four faces. Incident notes that mix them stay vague. MCP closes three of them and leaves the fourth to you.

| Face | What broke | What MCP standardizes |
|---|---|---|
| Tool schemas | One vendor's functions vs another's plugins | One list and call shape |
| Data access | A custom adapter per files, SQL, and HTTP | Resources, and tools that read |
| Orchestration glue | Each graph re-binds the same tools | Hosts share servers |
| Control plane | Allow-lists buried in prompts | The calls are inspectable. The policy is still yours |

* **Who may call**: IAM, a human approval, an allow-list. The protocol can show what was listed and what was called. It does not write the rule.
* **Where the key lives**: A secret store. The model should not see the key as a tool argument.
* **Whether the call was a good idea**: Evaluation and grounding. A successful JSON-RPC round trip can still be the wrong action.
* **How teams find and version tools**: A registry. MCP gives you a server. It does not give you a catalog.

A real internal API still needs an adapter behind the server. The protocol removed the per-host rewrite of the schema. Someone still maps your service's errors, auth, and payloads onto the primitives.

## 3. Host, client, server, service

The cable is the protocol. The laptop socket is the client. The charger is the server. Wall power is the service. Your agent is one kind of host: the app that holds clients.

* **Host**: The process a person or an orchestrator started. A desktop app, an IDE, a Python `Runner`, a platform worker. The host owns the UX, the permission prompts, and which servers are switched on.
* **Client**: The library inside the host that speaks JSON-RPC to one server. Four servers means four client sessions.
* **Server**: A process or an HTTP endpoint that advertises tools, resources, and prompts, and implements the calls.
* **Service**: GitHub, a folder, Postgres, a search API, your journal list. The server is an adapter in front of it.

On connect, client and server exchange capabilities. The client then **lists** tools, and maybe resources and prompts. The host stuffs those schemas into the model context. A chatty server is a prompt tax. Listing is not free.

A timeout belongs to one box. A hung `npx` is a server process that never became ready. An HTTP 401 from GitHub is the service. Tools that never appear in the trace are a client lifecycle or handshake bug. Tools that were listed, after which the model invented a URL, are a persona bug: the schemas were in context, and the instructions never required the call. That last one is not an MCP bug.

Servers can be composed. Someone can wrap your agent as a server another host calls. Do that after one server runs cleanly under the Inspector, not before.

## 4. Tools, resources, and prompts

A server may expose three primitives. They are not three spellings of `def`.

```
  list_tools / call_tool            -->  ACT
  list_resources / read_resource    -->  READ
  list_prompts / get_prompt         -->  SEED a persona fragment
```

1. **Tools**: Verbs. The agent calls them to do something, or to fetch with arguments: `record_event`, `get_research_sources`, `create_issue`. Each has a name, a description, a JSON-ish argument schema, and a result. Most agent hosts consume tools first. Some consume only tools. Use a tool when the call takes arguments or changes the world. A mail-sending tool is a mutation with a schema. It is not "just another function."

2. **Resources**: Nouns the host can read, and sometimes subscribe to. Files, configs, a database view, a URI. "Open this document" fits. A procedure with side effects does not. Use a resource to ground a turn or to show context in a desktop host. Forcing an argument-heavy action into a resource drops the schema that would have constrained it.

3. **Prompts**: Reusable instruction templates the server authors, such as "review this diff in our style." A host can offer them as slash-commands or as a starting fragment of a persona. They travel with the capability. They are not the agent's whole persona. A template that is versioned with the server should be the copy the host loads. A second, divergent copy pasted into `instructions` will drift.

A client that only implements tools pushes authors to wrap "read this file" and "here is a prompt" as tools so they show up. That workaround makes them visible. It also bloats the tool list (token cost, worse selection) and lies about side effects. If you control the host, implement the primitive. If you do not, name the stand-in honestly (`read_*` next to `run_*`) and document that it does not mutate. Empty primitives are not a score. Expose the ones this server actually has.

The description the Inspector shows is the description the model uses to choose. The docstring on `@mcp.tool()` is a prompt: when to call, what the arguments mean, what comes back, what empty means. "Provides a list" is weak. "Return the allow-listed sources; call this before planning; do not invent others" is operational.

## 5. Where the process lives

Transport follows the trust boundary. The protocol will carry a dangerous tool over either wire.

```
  LOCAL STDIO     host spawns the server on pipes
                  one caller, one lifetime, no port

  LOCAL SSE       server already listening on localhost
                  several agents can attach

  REMOTE          SSE/HTTP to a cloud host
                  auth, latency, a shared service

  HYBRID          secrets and files on STDIO
                  shared SaaS on remote SSE
```

1. **Local STDIO**: The host spawns the server, talks JSON-RPC on the child's stdin and stdout, and kills it when the session ends. No listen port. Low latency. The OS process boundary is the auth, plus whatever the host asked the human. Strictly one parent and one child. A second agent cannot attach to that pipe. Right for a desktop, a CLI, a test, and "this Python file is the server." Wrong for a shared cache: a new subprocess is a new empty `_journal = []`.

2. **Local SSE**: You start the server yourself (`mcp run -t sse ...`). It listens. Agents connect as HTTP clients. Several agents can share one process and one in-memory store. You now own a port, a health check, and the question "is it up?" Right when startup is slow or several agents must see the same journal. Wrong when you did not want shared mutable state.

3. **Remote**: Someone else owns the process. You own the URL, TLS, auth, and timeouts. Community SaaS servers and an internal platform endpoint show up this way. A slow remote tool stalls the agent loop. Timeouts are host policy.

4. **Hybrid**: High-blast-radius or high-privacy capabilities stay on STDIO next to the agent: files, secrets, scripts that touch production-adjacent state. Shared, boring capabilities go to remote SSE: search, calendar, GitHub. One Compose file that puts every server on SSE, including the one that can delete the workspace, "because we standardized on HTTP," ignores the trust boundary.

STDIO pays a spawn cost every run. SSE pays an always-on server. Desktop jobs and many CI jobs are correctly STDIO. A shared tool farm is correctly SSE or HTTP. "SSE means production" is a slogan, not a rule.

## 6. Which layer the server feeds

A server ships a capability to whichever layer needs it. Parking every MCP server in "tools" underuses the socket. When you add a server, name the layer. If you cannot, you cannot write the guardrail or the eval.

1. **Persona**: Prompt templates that seed a role. A fragment, such as an `incident_commander` prompt that travels with the server. The whole persona, including host-specific policy, stays in the agent's instructions. Two copies of the same policy will diverge.

2. **Tools and actions**: GitHub `create_issue`, a Slack post, allow-listed HTTP, the research-sources list. Classify read versus mutate. A tool that sends mail is an irreversible action that happens to have a schema.

3. **Reasoning and planning**: A server that does not touch the world and instead gives the agent a structured scratchpad. Sequential thinking is the usual example. The call still shows up as a tool span. The server makes the scratchpad inspectable and reusable. The planning method itself is still yours to design.

4. **Knowledge and memory**: Filesystem, Drive, databases, a graph, `load_journal`. The protocol carries "read this store" and "write that store" the same way it carries "create issue." Retrieval quality is a separate problem. The socket only moves the read and the write.

5. **Evaluation and feedback**: A rubric, a grounding check, a critic, exposed so more than one agent can call it. Wrapping a scorer in FastMCP does not replace rubrics, traces, or a regression set. It only shares the scorer.

The one people park in the wrong layer is the scratchpad: it looks like a tool call, and it is really a reasoning aid. Treat it as a writer of world-state and you will "guard" a notepad while the mail tool ships with no approval.

## 7. Writing the server

The smallest honest server is `get_research_sources` moved into a FastMCP process. The decorator publishes a schema. It does not run the agent.

```
  @mcp.tool()
  def get_research_sources() -> list[str]:
      """Return the allow-listed sources. Call before planning."""
```

* **Function name**: The tool name the model sees.
* **Signature**: The argument schema. A default the schema dropped is a protocol bug, not an LLM bug.
* **Return annotation**: The result shape the host may advertise. Return JSON-serializable types. A dict where the host expected a list is a schema bug.
* **Docstring**: The only description at list time. Write it as a prompt.
* **Same code, two entrypoints**: FastMCP serves STDIO when a host spawns the file, and SSE when you `mcp run -t sse`. Claude Desktop and `MCPServerStdio` can share that file. Two implementations means you pointed at two servers.
* **Import time stays boring**: `FastMCP("Research Tools")` at module level is fine. Opening network connections or mutating files at import makes STDIO spawn slow and makes an SSE reload surprising.
* **Stdout is the STDIO pipe**: A print for debugging is a byte on the RPC stream. It corrupts the session. Log to stderr or to a file.

A test that calls the Python function can pass while the host advertises a different name, a missing docstring, or a dropped parameter. The thing you shipped is the schema.

## 8. The Inspector

The **Inspector** is a host that is not an LLM. It connects, shows the live tool list, lets you call with arguments you choose, and shows the JSON-RPC. That split is how you stop blaming the model for a broken pipe.

* **Reach for it when**: The agent "doesn't see" a tool you registered. A call fails only inside the model loop, and you want the same arguments with no persona. STDIO and SSE disagree (spawn env, working directory, port). The agent runtime swallowed an initialize error.
* **The loop**: `mcp dev` on the absolute path of `server.py`, open the printed localhost URL, open Tools, run `get_research_sources`, confirm the names you expected.
* **Inspector succeeds, the agent never calls**: Persona, tool-list size, or sampling. The schema is fine. Stop editing the server.
* **Inspector fails**: Server, transport, or schema. A missing docstring, a default the schema dropped, or a dict where the host expected a list.
* **The host never connects**: Command path, `npx`, environment, or working directory. The model will otherwise improvise a tool result in prose.
* **Absolute paths**: A relative path is how a desktop app and a `Runner` launched from another directory load different files that share a name.

The cheapest experiment is one Inspector call with the same command line the SDK spawns. If that returns the three source names, stop editing the server and read the persona and the trace.

## 9. STDIO and SSE

Two transports show up throughout this chapter. Others exist (plain HTTP, stdio over SSH). Master these two. Switching in the Agents SDK is a constructor change, `MCPServerStdio` or `MCPServerSse`, plus how the process is started. Tool names should not change. If they do, you pointed at a different server.

1. **STDIO**: The client launches the server as a subprocess. JSON-RPC rides the child's stdin and stdout. No listen port. No extra auth on the wire. One parent, one child. Fits a desktop, a CLI, a test, `docker run -it`, and "run this file." Hurts when state must be shared across agents, when the host is remote, or when the server must outlive a single run. Measure spawn cost: you pay it every run.

2. **SSE (HTTP plus server-sent events)**: The server is already listening. The client opens a URL (`http://localhost:8000/sse` in the usual local example). Several clients can attach. Fits a local shared server, a container with a port, and a remote deployment. Hurts because you now own bind address, path, health, and timeouts. Auth is yours. NAT and proxies eat long-lived streams. You pay an always-on process instead of a spawn.

```
  STDIO:   host spawns ----pipes----> server    lifetimes coupled
  SSE:     server runs  <----HTTP----  host      lifetimes decoupled
```

`localhost` from inside a container is not your laptop. The URL is a DSN: one documented value per environment. FastMCP defaults to STDIO when spawned and to SSE when run with `-t sse`. A port hardcoded in three files will not stay that DSN.

## 10. A desktop host and an agent host

A desktop app will load your server and show a permission prompt. That is a useful test of the schema. It is a different execution model from an agent runtime. The modal is a security UX. Coding agents sometimes ask before a destructive call. Assistants sometimes auto-run a search. The modal is not the taxonomy.

```
  ASSISTANT HOST (typical desktop chat)
    user turn -> a few tool calls -> answer -> STOP
    the human schedules the next goal

  AGENT HOST (Runner)
    goal -> tool -> observe -> plan -> tool -> ...
    until a stop, a budget, or a guardrail
    no human click on each hop
```

* **Desktop success**: The schema loaded and a one-shot call returned.
* **Agent success**: Sense, plan, act, and learn still make sense across many calls. Idempotent tools, a clear "already recorded" result, and a persona that says when to stop matter once nobody clicks.
* **Errors are strings to the agent**: A desktop user sees a toast. The model sees an observation and may retry until the budget dies. Return a structured error.

### Watch for

* "It works in the desktop app" used as evidence the server is safe under `Runner.run` for a twenty-step goal. The desktop run stopped. The agent run loops, double-writes, or never stops.
* A tool that is not idempotent. The assistant clicked once. The agent may call `record_event` again because the observation did not say "already recorded."
* Debugging only by editing the prompt and rerunning. Inspector-versus-agent is the split that tells you which layer moved.

## 11. Attaching STDIO from the agent

The agent process owns the server process. An **action** is anything the agent does to finish a goal: a tool call, a resource read, or a handoff. MCP is how many of those actions get hosted. The runtime lists tools, puts schemas into the completion, executes the call, and feeds the observation back. The model does not speak MCP. The span you already know how to read is still the span.

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

* **`async with`**: The session lifetime is the child's lifetime. Done correctly, you do not leak `mcp` processes.
* **Absolute script path**: The working directory cannot silently pick another file.
* **Instructions still matter**: The server can list tools. The persona still has to say to use them.
* **Async runner**: STDIO and RPC wait. A naive sync story fights that wait.
* **Stale descriptions**: You edited the server and the agent still shows the old docstring. A spawned process loaded bytecode, or a different path. Confirm `SCRIPT`, kill strays, and point the Inspector at the same command the SDK runs.
* **Memory dies with the child**: An in-memory global resets when the process exits. A pure function such as an allow-list wants that. A journal does not. STDIO means the diary dies with the run unless the service persists it (a file, a database) instead of a module-level list.

## 12. Attaching SSE, and other people's servers

When spawn is the wrong topology — slow startup, several agents, a journal that must stay alive — run the server as HTTP and attach.

```
  # server process
  mcp run -t sse tracker_mcp.py

  # agent process
  async with MCPServerSse(
      name="SSE Python Server",
      params={"url": "http://localhost:8000/sse"},
  ) as research_server:
      ...
```

Same instructions, same tool names, a different connection. That is the point of the swap.

* **Start order**: The agent is not the process supervisor. A down URL is a client error, not a failed spawn.
* **Shared state**: Two agents, one SSE server, one `_journal`. That is coordination, or it is tests contaminating each other. A global list without a lock is a demo once more than one client connects.
* **Fan-out is why you picked SSE**: Multiple clients mean you have a concurrency story on the server.
* **Ops is the real cost**: The constructor swap is small. Environment, secrets, network, and state are not. STDIO remains the right default for many single-agent runs.

You should not hand-wrap GitHub in week one. Reference and community servers already exist: filesystem, sequential thinking, Drive, calendar, search, and a long tail installed with `npx`.

* **Read the list in the Inspector first**: Fifteen filesystem verbs will swamp a small persona. The short-list rule survives the protocol. Capability is not the number of MCP rows in a config file.
* **Pin versions**: `npx -y @scope/pkg` with no version is a supply-chain lottery and a behavior lottery.
* **Scope filesystem roots**: A server that can read a home directory is an exfiltration primitive. Pass an explicit directory. A server that can read a secrets volume is the same primitive with a worse root.
* **Name the layer while you attach it**: Sequential thinking is reasoning. Filesystem is usually tools plus knowledge. A calendar write is an irreversible tool.
* **Know which credential story you are in**: A token in a desktop config file is a different threat than a token in a platform secret store.
* **Configure before you fork**: A community server that is almost right gets a flag. A fork misses protocol updates.

Adding five `npx` servers "to look capable" spends the window on schemas. The model then calls nothing useful. Minimum servers for the goal.

## 13. Hosting the functions you used to import

The move you repeat at work: stop registering in-process functions on the `Agent`, and host them. The teaching story is a time tracker. `record_event` appends. `load_journal` reads. First both live in the agent file. Then they move.

```
  BEFORE                         AFTER
  agent.py                       tracker_mcp.py
    record_event                   @mcp.tool record_event
    load_journal                   @mcp.tool load_journal
    Agent(tools=[...])             _journal = []   # now a question
         |                              |
         v                              v
    Runner                         Agent(mcp_servers=[...])
```

* **Reuse**: A second agent, a desktop host, or an IDE loads the journal tools without importing your app.
* **Separation**: Persona and sampling stay in the host. I/O stays in the server. You can evaluate them apart.
* **One file, two transports**: STDIO today, SSE tomorrow, without a second implementation.
* **State becomes a decision**: A hidden global in the agent module was pretending not to be a database. Once the journal is a service, you say where it lives. In-memory is allowed in a lab. STDIO wipes it at the end of the run. A long-lived SSE process keeps it until restart. Anything real fronts a file or a database, with a concurrency story. MCP gives you a door. It does not give you durability.
* **Discovery replaces import**: The constructor drops `tools=[functions]` and gains a server session. A tool that "disappears" is a list or handshake failure.
* **Rewrite the docstring at the boundary**: Across a protocol, say the arguments, the idempotency rule (duplicate names append, or they reject), and whether `load_journal` must be called before `record_event`. Those sentences travel with the tool to every host. Host-specific policy stays in `instructions`: "always load the journal first; record new events; summarize from the tool; do not invent."
* **Do not paste the argument list twice**: The schema already provides it. Duplicated schemas drift, the same way a prose copy of `output_type` drifts.
* **Delete the demo verb**: A function that existed to print should not become a networked tool.
* **Keep a private function only while it is private**: Native function tools and MCP servers can sit on one agent. Do that for a tiny tool no other process needs. The moment a second agent needs it, host it. Leaving the old import "as a backup" keeps two schemas and invites a double write.

Remote consumption adds three host duties. Timeouts and retries are yours: a hung remote tool is an agent-loop failure. Auth headers belong on the client session, not in arguments the model can see. Schema drift (their server updated, your persona still describes old arguments) is cheaper to catch with the Inspector against the live server, read-only tools first, than to guess from a writeup.

Local consumption still has a threat model. A filesystem server is full access to whatever root you passed. The STDIO command in tests should be the command you believe the desktop host runs.

### Watch for

* `_journal = []` shipped as if it were durable. Predict it: three events in one STDIO run are visible inside that run; the next `Runner` process sees an empty list; one long-lived SSE process shows all events to both agents, until restart. None of those is a database.
* Instructions that copy the server's whole API. The server changes, the prose lies. Instructions say intent. Names of tools match the schema.
* An agent that still imports `record_event` after the server exists. You failed to separate I/O from the host. Schema drift and a double write are the bugs that follow.

## Check yourself

1. Draw three hosts and four services with a private plugin per pair, then draw the same picture with MCP. Where does the work you still pay for live? Name one internal API that would still need an adapter behind the server.
2. A timeout fires. For each case, name the first box you inspect (host, client, server, service, or persona): (a) `npx` never started, (b) GitHub returns 401, (c) tools never appear in the trace, (d) tools are listed and the model invents a URL. Why is (d) not an MCP bug?
3. Your IDE client does not support resources, so you expose `read_handbook` as a tool. What did you gain, what did you pay in the tool list, and when would you unwind the workaround?
4. A journal server uses `_journal = []`. What does it contain after three events in one STDIO run, after a second `Runner` process, and under one long-lived SSE server with two agents? Which topology is a demo, which is a shared store, and what is still missing for durability?
5. A filesystem server runs on SSE in a cluster "for consistency," and the process can read a secrets volume. Which deployment rule did that break, and which transport fits that server?
6. Map each to a layer: GitHub `create_issue`, a sequential-thinking server, a Drive read, a rubric scorer, and a prompt template named `incident_commander`. Which one is most often parked in the wrong layer, and what fails when you do that?
7. The Inspector can call `get_research_sources`. The agent never does. Name two persona or sampling causes and one MCP cause (handshake, path, or transport). What is the cheapest experiment that distinguishes them?
8. Why is "it works in a desktop chat" weak evidence that the same server is safe under `Runner.run` for a twenty-step goal? Answer with the loop, not with the permission modal.
9. You move in-process tools to MCP, and the agent still imports the functions as a backup. What did you fail to separate, and which bug becomes likely: schema drift, or a double write?
10. A teammate says the org tool service means you can skip learning servers. Which parts must an agent author still understand (list, call, transport, observation), and which parts sit outside the protocol (credentials, registry, policy)?
