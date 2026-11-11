# Appendix A: Setting up the sample code

The companion repo holds one folder of samples per chapter, built against OpenAI's tools and APIs. Clone `https://github.com/cxbxmxcx/AI-Agent-Workflows` and work from that directory.

Use Python 3.11 or newer in its own virtual environment (`python -m venv` on Windows, `python3 -m venv` on macOS or Linux, then activate it) so these packages stay off the rest of the machine. You can also point the editor at an interpreter you already have.

Install dependencies by launching a sample in the VS Code debugger, which installs them for you, or with `pip install -r requirements.txt` from the repo root while the environment is active. Copy the example env file to `.env`, set `OPENAI_API_KEY`, and keep that file local. It is a credential, and it belongs out of version control.

Run a sample from the repo root, for example `python chapter_02/01_first_agent.py`. An authentication error almost always means the key file is missing, in the wrong place, or stale after a rotation. An import error almost always means the wrong interpreter is active, or the requirements landed in a different environment.
