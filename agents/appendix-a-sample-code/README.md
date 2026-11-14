### **Appendix A Summary: Setting Up the Sample Code**

Appendix A of *AI Agents in Action, Second Edition* explains how to obtain the **companion repository**, isolate a Python runtime, wire an OpenAI API key, and confirm that chapter samples execute without environment drift.

---

#### **1. Cloning the Repository**

* **GitHub source**: Samples live in the **AI-Agent-Workflows** repository organized by chapter folder under a single project root.
* **Clone workflow**: A standard `git clone` followed by `cd` into the repo places all listings where subsequent setup commands expect them.
* **First-time order**: Cloning is the opening step; later sections assume commands run from that repository root.

---

#### **2. Creating a Python Environment**

* **Version requirement**: The book targets **Python 3.11 or newer**, with a dedicated virtual environment recommended over the system interpreter.
* **Platform activation**: Windows uses `python -m venv venv` and `venv\Scripts\activate`; macOS and Linux use `python3 -m venv venv` and `source venv/bin/activate`.
* **Interpreter selection**: VS Code users can pick a project interpreter via the command palette so the editor matches the environment used in the terminal.

---

#### **3. Installing Dependencies and Configuring the API Key**

* **Dependency install paths**: Pressing **F5 in VS Code** can auto-install requirements, or `pip install -r requirements.txt` installs pinned packages manually inside the active environment.
* **Secrets file**: Copy `.env.example` to `.env` and set **`OPENAI_API_KEY`** with a key from the OpenAI platform; never commit the `.env` file because it holds billing credentials.
* **Pinned requirements**: The root `requirements.txt` locks versions the chapter scripts were tested against, reducing “works on my machine” drift.

---

#### **4. Running Samples and Troubleshooting**

* **Running a sample**: Execute a chapter script from the repo root, e.g. `python chapter_02/01_first_agent.py`, mirroring the path pattern for other chapters.
* **Authentication failures**: Usually mean a missing `.env`, a wrong working directory, or a **rotated key** that was not updated locally.
* **Import failures**: Typically indicate the virtual environment is inactive or packages were installed into a different Python than the one running the script.
* **Healthy habits**: Keep interpreter, venv, and `.env` aligned, rely on `.gitignore` for secrets, and refresh the key in `.env` after any credential rotation.
