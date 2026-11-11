# Actions with the Model Context Protocol

MCP is how an agent reaches the world through one message shape. It is an open JSON-RPC standard, so a Python server and a TypeScript server look the same to the client. Before it, every model vendor wanted its own tool schema, every data source needed a custom adapter, agents had no shared way to offer their skills, and security was improvised per integration.

I keep three roles straight. The client connects and discovers. The server answers in the protocol. The service behind the server is the filesystem, the database, the API, or another agent. A server publishes three primitives, and control differs. Tools are actions the model chooses mid-conversation. Resources are data the user or the application attaches. Prompts are templates a person selects. Most of the agent work in this book is tools. Wrapping a resource or a prompt inside a tool is a workaround when the client only speaks tools. Keep the three distinct when the client can use them as intended.

Each server runs in one mode. Local STDIO is a child process on pipes: low latency, one caller, right for development and for anything that should stay off the network. Remote server-sent events over HTTP can serve many clients, sit behind a load balancer, and stream. A real agent often holds several of both in one session and treats them the same at the protocol level. Keep files and credentials local. Share search, chat, and other heavy APIs from a remote server. Inspect the live schema before you point an agent at it.

The same socket can feed every functional layer: actions against a repo or a chat app, a sequential-thinking aid, a database or vector store, a judge, or an approval gate. You can also wrap a whole agent as a server, so a workflow becomes a typed component another agent calls. Build the servers you need when you want tighter control over what an agent is allowed to do.

Agency raises the stakes. One wrong tool in a loop can chain. Allowlist what the agent may see, sandbox anything that touches the machine, check tool results before the next step, cap rate and spend, and keep a human on actions you cannot undo. Text that comes back from the outside is untrusted content in the prompt. Plan for deletes, leaked data, and a runaway bill.

- The integration lives on the server and is reused, which removes the per-agent glue.
- Moving between STDIO and HTTP is a client configuration change.
- An inspector and public reference servers let you debug the contract before you debug the agent.
- You still own authentication, versions, uptime, and the judgment of whether a call should happen.
