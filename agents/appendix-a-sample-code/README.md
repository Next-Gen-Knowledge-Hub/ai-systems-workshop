# Appendix A. Sample repository setup

Companion notes for **Appendix A** of *AI Agents in Action* (2nd edition, Micheal Lanham; Manning, 2026).

The listings are easier to trust when they actually run. This bench is one clone, one Python 3.11+ virtual environment, one `.env`, and one interpreter in the editor. System Python, a forgotten virtual environment, and a key left in shell history are three different machines. MCP listings also spawn a local Node process, so `node` and `npx` have to be on PATH before those files will start.

Official samples: [cxbxmxcx/AI-Agent-Workflows](https://github.com/cxbxmxcx/AI-Agent-Workflows).

## 1. What you are setting up

```
  git clone  -->  venv (3.11+)  -->  deps (F5 or pip)
                         |
                         v
                   .env  (OPENAI_API_KEY)
                         |
                         v
              python chapter_XX/some_sample.py
                         |
                         +-- traces in the vendor dashboard
                         +-- MCP samples also need Node/npx
```

* **One bench**: One virtual environment, one `.env` at the repo root, and the editor pointed at that same interpreter.
* **Where the listings live**: These study notes do not contain the book's listings. Clone the companion repo and work inside it.

## 2. Clone the repository

```bash
git clone https://github.com/cxbxmxcx/AI-Agent-Workflows.git
cd AI-Agent-Workflows
```

* **Git missing**: Install git from your OS, then retry the clone.
* **Plain clone**: Pulling `main` later is simpler from a plain clone. Fork only if you intend to keep local patches.
* **Repo root**: Skim the repo README once for the Python version badge and the chapter folders. Expect a layout like `chapter_02/01_....py`. Run from the repo root unless a comment says otherwise, so imports and `.env` loading resolve.

## 3. Create the Python environment

The companion code wants a current CPython. The repo badge and README win if they disagree with a remembered version. Treat **3.11+** as the default here. Run `python3 --version` before you create a virtual environment.

**Windows**

```text
python -m venv venv
venv\Scripts\activate
```

**macOS / Linux**

```text
python3 -m venv venv
source venv/bin/activate
```

* **Check the prompt**: After activate, the prompt should show the virtual environment. `which python` (or `where python` on Windows) should point inside `venv`, not at `/usr/bin/python3`.
* **Conda or pyenv**: You can skip `venv` if you already live in conda or pyenv. Point the editor at that interpreter. F5 or `pip` talking to a different Python than the terminal is the same failure either way.

## 4. Install dependencies

Two legitimate ways to get packages. Pick one per machine and stay with it until something breaks.

* **Path A — debug launch (F5)**: The companion repo can install requirements as part of starting a debug session. Use this when you want the book's `launch.json` and a breakpoint on the first agent run. Confirm the status-bar interpreter is the virtual environment you created.
* **Path B — pip**: With the virtual environment active:

```bash
pip install -r requirements.txt
```

`pip -V` should show a path under `venv`. If it shows a user or system site-packages, stop. That install is about to land in the wrong Python.

* **One tree**: If F5 and pip both ran, you can end up with two slightly different installs. When imports look wrong, delete `venv`, recreate it, and pick one path.
* **Editor interpreter**: Command Palette → "Python: Select Interpreter" → the virtual environment. A wrong interpreter is the usual `ModuleNotFoundError: agents` after a successful pip.

## 5. Add the OpenAI API key

Samples use the OpenAI Agents SDK and expect **`OPENAI_API_KEY`** in the environment.

* **`.env` at the repo root**: The repo ships a `.env.example` (or equivalent). Copy it to `.env` in the repo root and put a real key there. Do not commit `.env`. Do not paste keys into chat logs or into screenshots of traces.
* **Where the key comes from**: Create it in the OpenAI account dashboard. A compatible gateway may also need a base URL, and only when the sample documents that URL. Default listings talk to OpenAI.
* **Which value the process reads**: A key exported in the shell and a key in `.env` can both exist. The sample's `load_dotenv`, or the debugger env block, decides which one the process reads. When auth fails, print whether the variable is set (`len` only, never the value) before blaming the SDK.

## 6. Run a sample

From the repo root, with the virtual environment active:

```bash
python chapter_02/01_first_agent.py
```

* **Real filename**: Use the filename that is actually in the tree. Names move between printings.
* **What success looks like**: Terminal output from `Runner`, and, if tracing is on, a run in the OpenAI dashboard.
* **MCP samples**: Those listings spawn `npx` or a Node server. `npx: not found` means Node is missing from PATH. Reinstalling Python packages will not start that process.
* **Prove the bench first**: Get the earlier listings running (prompts, tools, then MCP) before you start the cognitive-agent sample.

## 7. Troubleshoot a failure

Match the symptom before you change the agent.

| Symptom | Look at first |
|---|---|
| `python` is 3.9 or missing | Install 3.11+; recreate the virtual environment |
| `No module named agents` | Interpreter is not the virtual environment; reinstall requirements |
| `OPENAI_API_KEY` / 401 | `.env` location, debugger env, key revoked |
| Hang on the first MCP sample | `node` or `npx` missing from PATH |
| F5 does nothing useful | Open the repo folder, not a parent; pick the interpreter |
| Weird Unicode or SSL on a corporate network | Proxy and the cert bundle |

* **429**: Rate limits and billing are account problems. A correct virtual environment still returns 429.

## 8. Keep the setup healthy

* **After `requirements.txt` changes**: `git pull`, then reinstall. Recreate the virtual environment rather than hoarding six-month-old wheels.
* **New import errors after a pull**: Run `pip list`. A sample may now import a package that was not installed.
* **Keys**: Rotate a key that leaked. Treat `.env` like a password file, and `chmod` it on Unix.
* **Node on the same machine**: MCP STDIO servers are local processes. Keep a current Node LTS on the same machine as this virtual environment.
* **Repo over these notes**: When the book and the repo disagree, the repo you just pulled is the runtime. These notes are the map.

## You are done when

- [ ] `AI-Agent-Workflows` is cloned and you work from its root
- [ ] A 3.11+ virtual environment is active, and `which python` points inside it
- [ ] Dependencies came from either F5 or `pip`, and you stuck to one path
- [ ] The editor interpreter is that same virtual environment
- [ ] `.env` at the repo root holds `OPENAI_API_KEY` and is not committed
- [ ] `python chapter_02/01_first_agent.py` (or the current first listing) prints `Runner` output
- [ ] An `npx: not found` failure points you at a Node install, not at pip

## Check yourself

1. You cloned the repo, opened a parent folder in the editor, and pressed F5. What fails, and which folder has to be the workspace root?
2. `pip install -r requirements.txt` succeeded, then the sample raises `No module named agents`. Which two commands show that the terminal Python and `pip` are outside the virtual environment?
3. Auth returns 401. A key is exported in the shell and a different key is in `.env`. What do you print, and what must that print omit?
4. The first MCP sample hangs and the shell says `npx: not found`. Why does another `pip install` leave that hang in place?
5. F5's installer and `pip` both ran, and imports disagree. What do you delete, and why do you pick a single install path when you recreate it?
6. `python3 --version` prints 3.9, and the new `venv` is 3.9 too. Which install did you skip, and what do you recreate after fixing it?
