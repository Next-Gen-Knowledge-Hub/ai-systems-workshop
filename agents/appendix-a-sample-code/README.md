# Appendix A. Sample repository setup

Companion notes for **Appendix A** of *AI Agents in Action* (2nd edition,
Micheal Lanham; Manning, 2026).

The Agents track is easier to believe when the listings run. This folder is
the **lab bench**: clone the companion repo, pin a Python, install deps,
put a key in a file that never gets committed, run a chapter sample, and
keep the bench from rotting. Skip it and every later "just run listing
3.x" becomes a weekend of interpreter archaeology.

This is Python plus secrets plus how you launch files. Many MCP samples
still need Node; finish Appendix B before those listings.

Official samples:
[cxbxmxcx/AI-Agent-Workflows](https://github.com/cxbxmxcx/AI-Agent-Workflows).

## What you are setting up

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

The one sentence: **one venv, one `.env`, one interpreter in the editor**
— mixing system Python, a forgotten venv, and a key in your shell history
is how "it works on my machine" becomes "it works on neither."

## Clone and land in the right tree

Clone the companion repo and work *inside* it. Workshop notes in
`ai-systems-workshop` are study material; they do not contain the book's
listings.

```bash
git clone https://github.com/cxbxmxcx/AI-Agent-Workflows.git
cd AI-Agent-Workflows
```

If `git` is missing, install it from your OS, then retry. Fork only if you
intend to keep local patches; pulling `main` later is simpler from a
plain clone.

Skim the repo README once for Python version badges and chapter folders.
Expect a layout like `chapter_02/01_....py` — run from the **repo root**
unless a comment says otherwise, so imports and `.env` loading resolve.

## Python environment

The companion code wants a **current CPython** (the repo's badge and
README win if they disagree with folklore; treat **3.11+** as the
workshop default). `python3 --version` before you invent a venv.

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

Your prompt should show the venv. `which python` (or `where python` on
Windows) should point **inside** `venv`, not at `/usr/bin/python3`.

If you already live in conda or pyenv, you can skip `venv` — then you
**must** point the editor at *that* interpreter. The failure is
identical either way: F5 or `pip` talking to a different Python than the
terminal.

## Dependencies: debugger path vs pip path

Two legitimate ways to get packages. Pick one per machine and stick to it
until something breaks.

**Path A — VS Code / Cursor debug (F5).** The companion repo is set up so
a debug launch can **install requirements as part of starting**. Useful
when you want the book's launch.json and a breakpoint on the first
agent run. Confirm the status bar interpreter is the venv you created.

**Path B — pip yourself.**

```bash
pip install -r requirements.txt
```

Run this only with the venv active. `pip -V` should show a path under
`venv`. If it shows a user or system site-packages, stop; you are about
to pollute the wrong Python.

If F5 and pip both ran, you can get **two slightly different trees**.
When imports look cursed, delete `venv`, recreate, pick **one** path.

Editor: Command Palette → "Python: Select Interpreter" → the venv. Wrong
interpreter is the number one "ModuleNotFoundError: agents" after a
successful pip.

## OpenAI API key (and what not to do)

Samples use the OpenAI Agents SDK and expect **`OPENAI_API_KEY`** in the
environment. The repo ships a `.env.example` (or equivalent). Copy it to
`.env` in the **repo root** and put a real key there. Do not commit
`.env`. Do not paste keys into chat logs or screenshots of traces.

Create a key in the OpenAI account dashboard. If your org uses a
compatible gateway, you may also need a base URL — only if the sample
documents it. Default listings talk to OpenAI.

Load order that surprises people: a key exported in the shell **and** a
`.env` file. Know which one the process actually reads (the sample's
`load_dotenv` vs the debugger env block). When auth fails, print whether
the variable is set (`len` only, never the value) before blaming the SDK.

## Running samples

From the repo root, with venv active:

```bash
python chapter_02/01_first_agent.py
```

(Use the real filename from the tree; names move between printings.)
You should see terminal output from `Runner` and, if tracing is on, a
run in the OpenAI dashboard.

MCP listings will spawn `npx` or a Node server. If those fail with
`npx: not found`, finish the Node setup in Appendix B. That is expected,
not a pip bug.

Do not start on the cognitive-agent sample until earlier chapter
listings (prompts, tools, MCP) run clean. The workshop order exists so
the bench is proven.

## Troubleshooting

When something fails, look at the symptom first, then the usual cause.

| Symptom | Look at first |
|---|---|
| `python` is 3.9 or missing | Install 3.11+; recreate venv |
| `No module named agents` | Interpreter ≠ venv; reinstall reqs |
| `OPENAI_API_KEY` / 401 | `.env` location, debugger env, key revoked |
| Hang on first MCP sample | Node/npx (Appendix B) |
| F5 does nothing useful | Open the **repo** folder, not a parent; pick interpreter |
| Weird Unicode / SSL on corp net | Proxy, cert bundle; not "the book is wrong" |

Rate limits and billing are **account** problems. A perfect venv will
still 429.

## Keeping the setup healthy

- Recreate the venv when `requirements.txt` changes upstream (`git pull`
  then reinstall). Do not hoard six-month-old wheels.
- `pip list` after a pull if a sample starts ImportErroring on a new
  package name.
- Rotate keys if they leaked; treat `.env` like a password file
  (`chmod` on Unix).
- Keep Node LTS in spec for MCP on the **same machine** as this venv —
  STDIO servers are local processes.
- When the book and the repo disagree, **the repo you just pulled** is
  the runtime truth; these notes are the map.

## You are done when

- [ ] `AI-Agent-Workflows` is cloned and you work from its root
- [ ] A 3.11+ venv is active; `which python` points inside it
- [ ] Dependencies installed via **either** F5 **or** `pip` (one path)
- [ ] Editor interpreter matches that venv
- [ ] `.env` holds `OPENAI_API_KEY` at the repo root and is not committed
- [ ] `python chapter_02/01_first_agent.py` (or the current first listing)
  prints Runner output
- [ ] You know where to go when `npx: not found` appears (Appendix B)
