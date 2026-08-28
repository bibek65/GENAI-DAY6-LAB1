# Day 06 — Agentic CI/CD with LangGraph

## Story
> "Devraj's pipeline failed at 2am. The agent read the log, found the past fix, drafted the PR — and a human reviewed it before anything merged."

**Duration:** 1.5 hours
**Local model:** `llama3.2:3b` + `nomic-embed-text`
**Framework:** LangGraph (`StateGraph`, `ToolNode`, `tools_condition`)
**Prereqs:** Day 05 `ci_knowledge_base` on disk at `../day5/ci_knowledge_base`

---

## The Problem

The Day 05 RAG system can find similar failures. That's retrieval.

But retrieval alone doesn't fix anything. Someone still has to:
- Read the log
- Search the knowledge base
- Form a root cause
- Write a PR description

At 2am, that someone is Devraj. Every time.

---

## The Solution — After Day 06

Same CI failure. Same Day 05 knowledge base. Now a LangGraph agent drives the workflow:

```
1. read_log_file("ci_failure_gha.log")
   → 1233 bytes of npm ERESOLVE errors

2. search_knowledge_base("npm error ERESOLVE unable to resolve dependency tree")
   → [similarity: 0.76] [nodejs]
     Fix: add --legacy-peer-deps or upgrade the shared package

3. analyze_root_cause()          ← no arguments needed
   → { "error_type": "dependency_conflict", "root_cause": "...",
       "fix_command": "npm install --legacy-peer-deps",
       "severity": "high", "confidence": "high" }

4. draft_pr_description()        ← no arguments needed
   → "## CI Failure — Automated Root Cause Analysis\n..."
```

Two files, four tools:

| File | What it contains |
|------|-----------------|
| `ci_tools.py` | The 4 tools + shared state. Reuses Day 05's `rag_tool.py` for retrieval |
| `lab1_ci_agent.py` | The LangGraph graph, the agent node, and the reliability controls |

---

# The Big Idea of Day 06

> **The LLM is probabilistic.**
> **The workflow should be deterministic and controlled by code.**

This is the message of the whole session. Everything below is an illustration of it.

You are not just learning "here is LangGraph." You are learning **how to build a reliable
agentic workflow around an imperfect local model** — which is the situation every team
is actually in.

---

## The Architecture

```
                 ┌──────────────────┐
                 │      AGENT       │
                 │    Llama 3.2     │
                 └────────┬─────────┘
                          │
                    LLM decides
                          │
                          ▼
                 ┌──────────────────┐
                 │ tools_condition  │
                 └────────┬─────────┘
                          │
                 ┌────────┴────────┐
                 │                 │
          tool requested       no tool
                 │                 │
                 ▼                 ▼
          ┌────────────┐          END
          │  ToolNode  │
          └─────┬──────┘
                │
                ▼
             Tool runs
                │
                ▼
             AGENT
                │
               ...
```

The loop keeps cycling `agent → tools → agent` until the LLM stops requesting tools.
Then `tools_condition` routes to `END`.

---

## Who Controls What

This is the distinction to hammer home.

### 1. The LLM controls *what it wants to do*

```
"I need to read the log."
        ↓
   read_log_file
```

It only **proposes**. It never executes anything.

### 2. LangGraph controls *where execution goes*

```
tool requested?
    ↓
YES → ToolNode
NO  → END
```

Routing is a function of state, not of model opinion. `tools_condition` inspects the
message for `tool_calls` and picks an edge. Same state, same route, every time.

### 3. Your Python code controls *what is actually allowed to happen*

For example:

```python
if len(response.tool_calls) > 1:
    response.tool_calls = response.tool_calls[:1]
```

Three lines that sit between "the model asked" and "the tool ran."

---

## Reliability Controls Around an LLM-Driven Workflow

The core loop is simple:

```
LLM
 │
 │ decides next action
 ▼
LangGraph
 │
 │ routes execution
 ▼
ToolNode
 │
 │ executes trusted Python tool
 ▼
Result
 │
 ▼
LLM
```

Your safeguards sit *around* that loop:

```
                    LLM
                     │
             ┌───────┴────────┐
             │ validation      │
             │ normalization   │
             │ one-tool limit  │
             └───────┬────────┘
                     │
                     ▼
              tools_condition
                 │         │
              tool        done
                 │         │
                 ▼         ▼
              ToolNode    END
```

These are not hacks and not LangGraph limitations. They are
**reliability controls around an LLM-driven workflow** — deliberate workflow design.

| Control | Where | What it prevents |
|---------|-------|------------------|
| One tool call per turn | `lab1_ci_agent.py` — `agent_node` | Model firing all 4 tools at once, breaking the dependency chain |
| Argument normalization | `lab1_ci_agent.py` — `_normalize_args` | Invented parameter names (`q` instead of `query`) failing validation silently |
| `_LAST` intermediate state | `ci_tools.py` | Forcing the model to echo a 1 KB log through JSON arguments |
| Log path fallback | `ci_tools.py` — `read_log_file` | Hallucinated file paths returning an error the model then ignores |
| Type coercion | `ci_tools.py` — `_as_text` | A `dict` arriving where the tool declared `str` |
| Output extraction | `lab1_ci_agent.py` — `main` | Relying on a 3B model to echo 1 KB of markdown back verbatim |

---

### Control 1 — The one-tool limit

Without it, `llama3.2:3b` requests everything simultaneously:

```
LLM
 ├── read_log_file
 ├── search_knowledge_base
 ├── analyze_root_cause
 └── draft_pr_description
```

None of the later tools can use the earlier results, because none have run yet.
The model is guessing what the log says before reading it.

Instead, you enforce:

```
read
 ↓
result
 ↓
search
 ↓
result
 ↓
analyze
 ↓
result
 ↓
draft
```

```python
if len(response.tool_calls) > 1:
    response.tool_calls = response.tool_calls[:1]
```

**That's a workflow design decision, not a LangGraph limitation.**

---

### Control 2 — `_LAST`: the application holds the state

> "The model doesn't need to carry the entire log from one tool call into the next tool
> call. The application stores the intermediate results, while LangGraph manages the
> execution flow."

```
read_log_file
      │
      ▼
_LAST["log"]
      │
      ▼
analyze_root_cause
```

and:

```
search_knowledge_base
      │
      ▼
_LAST["kb"]
      │
      ▼
analyze_root_cause
```

You are deliberately separating **workflow state / control** from **LLM decision-making**.

The payoff: the model can call `analyze_root_cause()` with **no arguments at all**. The
tool reaches for the real data itself. Fewer tokens for the model to get wrong.

---

### Control 3 — Argument normalization

`llama3.2:3b` reliably sends `{'q': '...'}` to a tool whose parameter is `query`. Pydantic
rejects the unknown key, the tool never runs, and the model ignores the error message.

`_normalize_args` drops unknown keys and remaps stray values onto free schema fields:

```
{'q': 'npm error ERESOLVE...'}  →  {'query': 'npm error ERESOLVE...'}
```

It prints a `🩹` line whenever it fires, so the repair is visible rather than magic.

---

## Setup — Before Labs Start

### Step 1 — Activate the virtual environment

> ⚠️ **Modern Python (3.11+) blocks direct pip installs into the system.**
> Always work inside a virtual environment.

**If you already created `workshop-env` on an earlier day, just activate it:**

Linux / macOS:
```bash
source ~/workshop-env/bin/activate
```

Windows (PowerShell):
```powershell
$HOME\workshop-env\Scripts\Activate.ps1
```

**If you have NOT created it yet, create it first:**

Linux / macOS:
```bash
python3 -m venv ~/workshop-env
source ~/workshop-env/bin/activate
```

Windows (PowerShell):
```powershell
python -m venv $HOME\workshop-env
$HOME\workshop-env\Scripts\Activate.ps1
```

Your prompt should now show **`(workshop-env)`** — this means the venv is active.

Confirm you are using the venv's Python, not the system one:

```bash
which python3      # Linux/macOS  → should print a path under ~/workshop-env
```

---

### Step 2 — Install the packages

```bash
pip install langchain-ollama langgraph chromadb
```

`langchain-core` and `requests` install automatically as dependencies.

Verify:

```bash
python3 -c "import langgraph, langchain_ollama, chromadb; print('deps ok')"
```

| Package | Used by |
|---------|---------|
| `langchain-ollama` | `ChatOllama` — the agent LLM |
| `langgraph` | `StateGraph`, `ToolNode`, `tools_condition` |
| `chromadb` | Day 05's `rag_tool.py` — the vector store |
| `requests` | The RCA call to Ollama, and Day 05's embedding call |

---

### Step 3 — Check the Ollama models

```bash
ollama list | grep llama3.2
ollama list | grep nomic-embed
```

| Model | Used for |
|-------|----------|
| `llama3.2:3b` | The agent's tool decisions **and** the root cause analysis |
| `nomic-embed-text` | Embedding the search query inside Day 05's `rag_tool.py` |

If either is missing:

```bash
ollama pull llama3.2:3b
ollama pull nomic-embed-text
```

---

### Step 4 — Check the Day 05 knowledge base

```bash
cd ~/Desktop/Personal/AI/day6
ls ../day5/ci_knowledge_base/chroma.sqlite3
```

**Expected:** the path prints back.

If missing:

```bash
cd ../day5 && python3 embed_failures.py && cd ../day6
```

> **Day 06 never writes to the knowledge base.** It only reads what Day 05 built.

> **Note:** `chromadb` and `nomic-embed-text` are never referenced by Day 06's own code.
> They arrive through the `from rag_tool import search_similar_failures` line in
> `ci_tools.py`. Day 06 inherited a dependency it never declared — exactly what happens
> when you reuse a module across projects.

---

# Lab 1 — CI Failure Analyzer with LangGraph ⏱️ 40 min

**Goal:** Build a 4-tool CI failure analyzer as a LangGraph agent, with explicit
reliability controls around an unreliable local model.

---

## Step 1.1 — Create the failure log

`ci_failure_gha.log`:

```
2024-01-15T02:14:03Z Run npm install
2024-01-15T02:14:04Z npm warn deprecated inflight@1.0.6
2024-01-15T02:14:07Z npm error code ERESOLVE
2024-01-15T02:14:07Z npm error ERESOLVE unable to resolve dependency tree
2024-01-15T02:14:07Z npm error
2024-01-15T02:14:07Z npm error While resolving: payment-service@2.4.1
2024-01-15T02:14:07Z npm error Found: react@18.2.0
2024-01-15T02:14:07Z npm error node_modules/react
2024-01-15T02:14:07Z npm error   react@"^18.2.0" from the root project
2024-01-15T02:14:07Z npm error
2024-01-15T02:14:07Z npm error Could not resolve dependency:
2024-01-15T02:14:07Z npm error peer react@"^17.0.0" from @company/shared-utils@1.2.3
2024-01-15T02:14:07Z npm error node_modules/@company/shared-utils
2024-01-15T02:14:07Z npm error   @company/shared-utils@"^1.2.0" from the root project
2024-01-15T02:14:07Z npm error
2024-01-15T02:14:07Z npm error Fix the upstream dependency conflict, or retry
2024-01-15T02:14:07Z npm error this command with --force or --legacy-peer-deps
2024-01-15T02:14:08Z npm error A complete log of this run can be found in:
2024-01-15T02:14:08Z npm error /home/runner/.npm/_logs/2024-01-15T02_14_07_432Z-debug-0.log
2024-01-15T02:14:08Z Error: Process completed with exit code 1.
```

**What actually broke:** `@company/shared-utils@1.2.3` declares a peer dependency on
`react@^17.0.0`, but the root project `payment-service@2.4.1` requires `react@^18.2.0`.
Since npm 7, peer dependencies are enforced strictly — no valid tree exists, so `npm install`
aborts with exit code 1.

---

## Step 1.2 — Create `ci_tools.py`

### Imports and Day 05 reuse

```python
# ci_tools.py

import os
import sys
import json
from typing import Union

import requests

from langchain_core.tools import tool

# ----------------------------------------------------------------------------
# Reuse the Day 05 RAG tool instead of rebuilding embeddings + ChromaDB here.
# ----------------------------------------------------------------------------

DAY5_DIR = os.path.join(
    os.path.dirname(
        os.path.dirname(os.path.abspath(__file__))
    ),
    "day5",
)

if DAY5_DIR not in sys.path:
    sys.path.insert(0, DAY5_DIR)

from rag_tool import search_similar_failures
```

> **Why this matters:** Day 06 owns *no* embeddings, *no* ChromaDB client, and *no* seed
> data. It imports one function from Day 05. One knowledge base, one source of truth.

### Configuration

```python
SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))

MODEL = "llama3.2:3b"

OLLAMA_GENERATE_URL = "http://localhost:11434/api/generate"

LOG_FILE = "ci_failure_gha.log"
```

### Shared state + coercion helper

```python
# Outputs of previous tool calls. Small local models cannot reliably
# echo a 1KB log back through JSON tool arguments, so later tools fall
# back to what the earlier tools actually produced.
_LAST = {
    "log": "",
    "kb": "",
    "analysis": "",
}


def _as_text(value: Union[str, dict, list, None]) -> str:
    """Tool args sometimes arrive as dict/list instead of str. Normalize."""

    if value is None:
        return ""

    if isinstance(value, str):
        return value

    return json.dumps(value, indent=2)
```

### Tool 1 — read the log

```python
@tool
def read_log_file(filepath: str = LOG_FILE) -> str:
    """
    Read a CI failure log file and return its contents.
    """

    resolved = (
        filepath
        if os.path.isabs(filepath)
        else os.path.join(SCRIPT_DIR, filepath)
    )

    # The LLM often invents a path. Fall back to the log that ships
    # with this lab instead of returning an error it will ignore.
    if not os.path.isfile(resolved):
        resolved = os.path.join(SCRIPT_DIR, LOG_FILE)

    print("\n🔧 TOOL CALL: read_log_file")
    print(f"   filepath = {resolved}")

    try:

        with open(resolved, "r") as file:
            content = file.read()

        _LAST["log"] = content

        print(f"   ✅ Read {len(content)} bytes")

        return content

    except FileNotFoundError:
        return f"Error: file not found at {resolved}"

    except Exception as exc:
        return f"Error reading file: {exc}"
```

**Reliability control:** the path fallback. The model has invented
`/var/log/ci_failure.log` before. Returning an error string just makes it hallucinate
log contents instead.

### Tool 2 — search the Day 05 knowledge base

```python
@tool
def search_knowledge_base(query: str) -> str:
    """
    Search the CI failure knowledge base for similar historical failures.
    """

    print("\n🔧 TOOL CALL: search_knowledge_base")
    print(f"   query = {query}")

    try:
        result = search_similar_failures(query)

    except Exception as exc:
        return f"Error searching knowledge base: {exc}"

    _LAST["kb"] = result

    matches = result.count("[similarity:")

    print(f"   ✅ Found {matches} similar failure(s)")

    return result
```

All the embedding and vector search happens inside Day 05's `search_similar_failures`.
This tool is a thin wrapper that logs and caches.

### Tool 3 — root cause analysis (a tool that calls an LLM)

```python
@tool
def analyze_root_cause(
    log_content: Union[str, dict, list] = "",
    past_failures: Union[str, dict, list] = "",
) -> str:
    """
    Analyze the CI failure and return a JSON root cause analysis.
    Call this with NO arguments — it automatically uses the log from
    read_log_file and the matches from search_knowledge_base.
    """

    print("\n🔧 TOOL CALL: analyze_root_cause")

    log_content = _as_text(log_content) or _LAST["log"]
    past_failures = _as_text(past_failures) or _LAST["kb"]

    if not log_content.strip():
        return "Error: no log available. Call read_log_file first."

    context = ""

    if past_failures:
        context = (
            "\n\nSIMILAR HISTORICAL FAILURES "
            "(previously resolved — reuse their fixes when the "
            "failure pattern matches):\n"
            f"{past_failures}"
        )

    prompt = f"""
You are a DevOps engineer analyzing a CI pipeline failure.

Analyze the CI log below.

Return ONLY valid JSON with exactly these keys:

error_type, root_cause, affected_file, fix_command, severity, confidence

Rules:

- error_type must be a short snake_case label.
- root_cause must quote the specific package names, versions and
  error codes found in the CI log below.
- affected_file must identify the file a human would edit to fix
  this. Never point at generated directories such as node_modules.
- fix_command must be a runnable shell command. If a similar
  historical failure below documents a fix that applies here,
  prefer that documented fix over any other option.
- severity must be low, medium, or high.
- confidence must be high when the log names an explicit error code
  and a similar historical failure was found.
- Do not invent package names or versions.
- Return only JSON.

CI FAILURE LOG:

{log_content}

{context}
"""

    try:
        response = requests.post(
            OLLAMA_GENERATE_URL,
            json={
                "model": MODEL,
                "prompt": prompt,
                "format": "json",
                "stream": False,
            },
            timeout=120,
        )
        response.raise_for_status()
        result = response.json()["response"]

    except Exception as exc:
        return f"Error running root cause analysis: {exc}"

    _LAST["analysis"] = result

    print("   ✅ Root cause analysis completed")

    return result
```

Three things to point out here:

1. **A tool can itself call an LLM.** The agent LLM decides *to analyze*; this nested call
   *does* the analyzing. Its JSON goes back into graph state as a `ToolMessage`.
2. **`"format": "json"`** forces syntactically valid JSON out of `llama3.2:3b`.
   It does **not** guarantee which keys appear.
3. **The retrieved history is an instruction, not decoration.** The prompt explicitly says
   *prefer that documented fix* — otherwise the model retrieves the right precedent and
   then ignores it.

### Tool 4 — draft the PR description

```python
@tool
def draft_pr_description(
    analysis_json: Union[str, dict] = "",
    past_failures: Union[str, dict, list] = "",
) -> str:
    """
    Generate a GitHub PR description from the root cause analysis.
    Call this with NO arguments — it automatically uses the output of
    analyze_root_cause and search_knowledge_base.
    """

    print("\n🔧 TOOL CALL: draft_pr_description")

    past_failures = _as_text(past_failures) or _LAST["kb"]

    if not analysis_json:
        analysis_json = _LAST["analysis"]

    if not analysis_json:
        return "Error: no analysis available. Call analyze_root_cause first."

    if isinstance(analysis_json, dict):
        analysis = analysis_json
    else:
        try:
            analysis = json.loads(analysis_json)
            if not isinstance(analysis, dict):
                analysis = {"root_cause": str(analysis)}
        except (json.JSONDecodeError, TypeError):
            analysis = {"root_cause": _as_text(analysis_json)}

    historical_context = ""

    if past_failures:
        historical_context = (
            "\n### Historical Context\n\n"
            f"{past_failures[:600]}\n"
        )

    pr_body = f"""## CI Failure — Automated Root Cause Analysis

### Error

**Type:** {analysis.get("error_type", "Unknown")}

**Severity:** {analysis.get("severity", "Unknown")}

**Confidence:** {analysis.get("confidence", "Unknown")}

### Root Cause

{analysis.get("root_cause", "Not determined")}

### Affected File / Config

`{analysis.get("affected_file", "Unknown")}`

### Proposed Fix

{analysis.get("fix_command", "# No fix command generated")}
{historical_context}

Generated by CI Failure Analyzer Agent — human reviewed before merge
"""

    print(f"   ✅ PR description generated ({len(pr_body)} characters)")

    return pr_body
```

**Reliability control:** the PR body is built by **Python string formatting**, not by the
LLM. The model supplies six short values; the document structure is deterministic. If a
key is missing you get `Unknown`, not a hallucinated section.

### Tool registry

```python
TOOLS = [
    read_log_file,
    search_knowledge_base,
    analyze_root_cause,
    draft_pr_description,
]
```

---

## Step 1.3 — Create `lab1_ci_agent.py`

### Imports, model, tool binding

```python
from langchain_ollama import ChatOllama
from langchain_core.messages import HumanMessage, SystemMessage, ToolMessage

from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import ToolNode, tools_condition

from ci_tools import TOOLS

MODEL_NAME = "llama3.2:3b"

model = ChatOllama(
    model=MODEL_NAME,
    temperature=0,
)

model_with_tools = model.bind_tools(TOOLS)
```

`bind_tools` sends each tool's name, docstring and JSON schema to the model so it knows
what it may request.

> **`temperature=0`** makes the agent's *decisions* deterministic — same conversation,
> same tool chosen. Note that the nested call inside `analyze_root_cause` does not set
> temperature, so it samples at Ollama's default `0.8`. That is why the tool *sequence*
> is identical every run while the *analysis text* varies.

### Reliability control — argument normalization

```python
TOOLS_BY_NAME = {tool.name: tool for tool in TOOLS}


def _schema_fields(tool):
    """Parameter names the tool actually accepts."""
    return list(tool.args_schema.model_fields.keys())


def _normalize_args(tool_name: str, args: dict) -> dict:
    """
    Small local models invent parameter names (e.g. `q` for `query`).
    Drop unknown keys and remap a stray value onto the first free
    parameter so the call validates instead of silently failing.
    """

    tool = TOOLS_BY_NAME.get(tool_name)

    if tool is None or not isinstance(args, dict):
        return args

    valid = _schema_fields(tool)

    kept  = {k: v for k, v in args.items() if k in valid}
    stray = [v for k, v in args.items() if k not in valid]
    free  = [n for n in valid if n not in kept]

    for name, value in zip(free, stray):
        kept[name] = value

    if kept != args:
        print(f"   🩹 Normalized arguments: {args} → {kept}")

    return kept
```

### The agent node — where the LLM decides

```python
def agent_node(state: MessagesState):
    """
    The LLM receives the conversation and decides:
        1. Call a tool, OR
        2. Return the final answer
    """

    print()
    print("=" * 70)
    print("🤖 AGENT NODE")
    print("=" * 70)

    response = model_with_tools.invoke(state["messages"])

    if response.tool_calls:

        # Small local models emit every tool at once, which breaks the
        # read → search → analyze → draft dependency chain. Keep only
        # the first call so each step sees the previous step's output.
        if len(response.tool_calls) > 1:

            print(
                f"⚠️  LLM requested {len(response.tool_calls)} tools at once — "
                f"keeping only the first."
            )

            response.tool_calls = response.tool_calls[:1]

        print("🧠 LLM requested tool:")

        for tool_call in response.tool_calls:

            print(f"   → {tool_call['name']}")

            tool_call["args"] = _normalize_args(
                tool_call["name"],
                tool_call["args"],
            )

            print(f"     Arguments: {tool_call['args']}")

    else:
        print("🧠 LLM returned a final response.")

    return {"messages": [response]}
```

**Both reliability controls live here**, in the gap between "the model asked" and
"the tool ran."

### The tools node — where trusted Python executes

```python
_tool_node = ToolNode(TOOLS)


def tool_node(state: MessagesState):
    """Run the requested tools and show what came back."""

    result = _tool_node.invoke(state)

    for message in result["messages"]:

        preview = str(message.content).replace("\n", " ")

        if len(preview) > 200:
            preview = preview[:200] + " …"

        print(f"   ↩︎  {message.name}: {preview}")

    return result
```

Wrapping `ToolNode` makes every result visible. Without this, a failed tool call is
silent — the model just quietly moves on with an error string it ignored.

### Build the graph

```python
builder = StateGraph(MessagesState)

builder.add_node("agent", agent_node)
builder.add_node("tools", tool_node)

builder.add_edge(START, "agent")

builder.add_conditional_edges(
    "agent",
    tools_condition,
    {
        "tools": "tools",
        END: END,
    },
)

builder.add_edge("tools", "agent")

graph = builder.compile()
```

Four lines of routing. That is the entire deterministic control flow:

```
START → agent → (tools_condition) → tools → agent → ... → END
```

### Run it

```python
def main():

    system_prompt = """
You are a CI failure analysis agent.

Important rules:

- Call exactly ONE tool per turn, then wait for its result.
- Never invent tool results or file paths.
- Always pass the previous tool's actual output into the next tool.
- Never skip a step.
- After draft_pr_description returns, reply with its output verbatim
  and call no further tools.
"""

    task_prompt = """
Investigate the CI failure and prepare a proposed GitHub PR description.

Follow these steps, one tool call per turn:

1. read_log_file with filepath="ci_failure_gha.log"
2. search_knowledge_base with a query built from the error text in the log
3. analyze_root_cause with NO arguments (it reuses steps 1 and 2)
4. draft_pr_description with NO arguments (it reuses steps 2 and 3)

Never copy log text or search results into tool arguments.
The tools already remember them.
"""

    result = graph.invoke(
        {
            "messages": [
                SystemMessage(content=system_prompt),
                HumanMessage(content=task_prompt),
            ]
        },
        {"recursion_limit": 25},
    )

    # The PR body is what draft_pr_description actually produced.
    # Don't rely on a 3B model to echo 1KB of markdown back verbatim.
    pr_body = None

    for message in result["messages"]:
        if isinstance(message, ToolMessage) and message.name == "draft_pr_description":
            pr_body = message.content

    print(pr_body if pr_body else "⚠️  draft_pr_description never ran.")


if __name__ == "__main__":
    show_graph()
    main()
```

> **`recursion_limit: 25`** caps the `agent → tools → agent` loop. Without it, a model that
> keeps requesting tools loops forever.

> **The final output is pulled from the `ToolMessage`, not the model's closing text.**
> A 3B model asked to "reply with its output verbatim" sometimes replies
> *"This is the final output."* instead. The data is already in graph state — read it there.

---

## Step 1.4 — Run Lab 1

```bash
cd ~/day6
python3 lab1_ci_agent.py
```

---

## Step 1.5 — Reading the output

The run prints the compiled graph, then one block per loop iteration.

| Line you'll see | What it proves |
|-----------------|----------------|
| `graph TD; __start__ --> agent; agent -.-> tools;` | The graph structure LangGraph compiled |
| `🧠 LLM requested tool: → read_log_file` | The LLM **decided** — nothing hardcoded the order |
| `🩹 Normalized arguments: {'q': ...} → {'query': ...}` | The model got the parameter name wrong and **code caught it** |
| `⚠️  LLM requested 4 tools at once — keeping only the first` | The one-tool limit firing |
| `✅ Read 1233 bytes` | A trusted Python tool executed |
| `✅ Found 1 similar failure(s)` | Real semantic retrieval against Day 05's ChromaDB |
| `↩︎  analyze_root_cause: {...}` | A tool that itself called an LLM, returning JSON into state |
| `🧠 LLM returned a final response.` | `tools_condition` routes to `END` — the loop terminates |

### Expected output (abridged)

```
======================================================================
🤖 AGENT NODE
======================================================================
🧠 LLM requested tool:
   → read_log_file
     Arguments: {'filepath': 'ci_failure_gha.log'}

🔧 TOOL CALL: read_log_file
   ✅ Read 1233 bytes
   ↩︎  read_log_file: 2024-01-15T02:14:03Z Run npm install …

======================================================================
🤖 AGENT NODE
======================================================================
🧠 LLM requested tool:
   → search_knowledge_base
   🩹 Normalized arguments: {'q': 'npm error ERESOLVE...'} → {'query': 'npm error ERESOLVE...'}

🔧 TOOL CALL: search_knowledge_base
   ✅ Found 1 similar failure(s)
   ↩︎  search_knowledge_base: [similarity: 0.76] [nodejs] Node.js build failed: npm ERR ERESOLVE …

======================================================================
🤖 AGENT NODE
======================================================================
🧠 LLM requested tool:
   → analyze_root_cause
     Arguments: {}

🔧 TOOL CALL: analyze_root_cause
   ✅ Root cause analysis completed

======================================================================
🤖 AGENT NODE
======================================================================
🧠 LLM requested tool:
   → draft_pr_description
     Arguments: {}

🔧 TOOL CALL: draft_pr_description
   ✅ PR description generated (1010 characters)

======================================================================
✅ WORKFLOW COMPLETED
======================================================================

## CI Failure — Automated Root Cause Analysis

### Error

**Type:** ERESOLVE

**Severity:** high

**Confidence:** high

### Root Cause

peer dependency conflict between react@18 and @company/shared-utils@1.2.3,
which requires react@17

### Affected File / Config

`/home/runner/project/node_modules/@company/shared-utils/index.js`

### Proposed Fix

npm install --legacy-peer-deps

### Historical Context

[similarity: 0.76] [nodejs]
Node.js build failed: npm ERR ERESOLVE unable to resolve dependency tree. …

Generated by CI Failure Analyzer Agent — human reviewed before merge
```

> **This is a real run, not an idealized one.** Root cause and fix are correct.
> `affected_file` is **wrong** — it should be `package.json`. Lab 3 explains why that
> particular field fails every time.

> Note `analyze_root_cause` and `draft_pr_description` are called with `{}` — **empty
> arguments**. `_LAST` supplied the real data. That is the separation of workflow state
> from LLM decision-making, visible on screen.

---

# Lab 2 (Bonus) — Break the One-Tool Limit ⏱️ 10 min

Comment out three lines in `agent_node`:

```python
# if len(response.tool_calls) > 1:
#     response.tool_calls = response.tool_calls[:1]
```

Run again.

### What you will observe

```
🧠 LLM requested tool:
   → read_log_file
     Arguments: {'filepath': '/var/log/ci_failure.log'}
   → search_knowledge_base
     Arguments: {'query': 'CI failure "Failed to connect to database"'}
   → analyze_root_cause
     Arguments: {'log_content': 'The log file contains: "Failed to connect to database"…'}
   → draft_pr_description
     Arguments: {'analysis_json': {'root_cause': 'Database connection issue'}}
```

All four at once — and every argument is **invented**. The model wrote what it *imagined*
the log said, because it hasn't read it yet. There is no database in this pipeline.

### What this teaches

The model is not being stupid. It is being **probabilistic**: asked to do four things, it
predicts all four tool calls in one pass because nothing in the token stream forces it to
wait.

**Waiting is a property of your workflow, not of the model.** Three lines of Python
restore it.

---

# Lab 3 (Bonus) — Where the Small Model Still Fails ⏱️ 10 min

Run `lab1_ci_agent.py` three times and compare the `analyze_root_cause` output.

The tool *sequence* will be identical every run. The *analysis* will not.

### The answer key

| Field | Correct value |
|-------|--------------|
| `error_type` | `dependency_conflict` |
| `root_cause` | `@company/shared-utils@1.2.3` needs peer `react@^17.0.0`, root project requires `react@^18.2.0` |
| `affected_file` | `package.json` |
| `fix_command` | `npm install --legacy-peer-deps` |
| `severity` | `high` |
| `confidence` | `high` |

### Observed failure modes

| Failure | Example | Why |
|---------|---------|-----|
| **Can't infer unstated facts** | `affected_file` → `node_modules/...`, or the npm debug log path | `package.json` **never appears in the log**. The model scans for something *shaped like a path* instead of reasoning about which file a human edits |
| **Red herrings** | listed `inflight@1.0.6` as a cause | That's a deprecation *warning* on line 2, unrelated to the failure |
| **Version transplant** | proposed `@company/shared-utils@13.0.0` | Took "upgrade to **version 13**" from the retrieved KB entry and stuck the number on a different package. RAG can *cause* this class of error |
| **Ignores negative constraints** | kept returning `node_modules` paths | The prompt says "never point at node_modules." Small models handle "don't" poorly |

### The lesson

The fields the model got **right** are the ones a retrieved document could anchor
(`fix_command`, `severity`). The field it got **wrong every single time** is the one
requiring inference from domain knowledge (`affected_file`).

> A 3B model can extract what's written and reuse what's retrieved.
> It cannot reliably infer what isn't stated.

This is exactly why the PR template ends with **"human reviewed before merge."**

---

# Day 06 Recap

| Lab | What you did |
|-----|-------------|
| Lab 1 | 4-tool CI agent on LangGraph, with reliability controls |
| Lab 2 (Bonus) | Removed the one-tool limit to observe the failure mode |
| Lab 3 (Bonus) | Measured where a 3B model's reasoning breaks down |

### File layout

```
~/day6/
├── ci_failure_gha.log      ← mock GitHub Actions failure
├── ci_tools.py             ← 4 tools + _LAST state (imports Day 05 RAG)
└── lab1_ci_agent.py        ← LangGraph graph + reliability controls

~/day5/
├── rag_tool.py             ← search_similar_failures()  ← imported by Day 06
├── embed_failures.py       ← seeds the knowledge base
└── ci_knowledge_base/
    └── chroma.sqlite3      ← 10 documents, read-only from Day 06
```

### What you have now

```
✅ Structured output           — Day 03
✅ Tool calling + LangGraph    — Day 04
✅ RAG + semantic search       — Day 05
✅ Agentic CI/CD + guardrails  — Day 06  ← new today
```

### Tool quick reference

| Tool | Input | Output |
|------|-------|--------|
| `read_log_file` | `filepath: str = "ci_failure_gha.log"` | raw log content |
| `search_knowledge_base` | `query: str` | similarity matches from Day 05 ChromaDB |
| `analyze_root_cause` | *none — reads `_LAST`* | JSON: error_type, root_cause, affected_file, fix_command, severity, confidence |
| `draft_pr_description` | *none — reads `_LAST`* | Markdown PR body with historical context |

### Golden Rules

1. **The LLM is probabilistic; the workflow must be deterministic.** Let the LLM decide, let code enforce.
2. **Never trust tool arguments.** Normalize names, coerce types, and keep real data in `_LAST`.
3. **One tool call per turn** when the chain is order-dependent. That's workflow design, not a framework limit.
4. **Make failures visible.** Print every `ToolMessage` — a silent tool error looks like success.
5. **Read results from state, not from the model's closing message.** The data is already in the graph.
6. **RAG before analysis.** `search_knowledge_base` must run before `analyze_root_cause`, and the prompt must *tell the model to use* what came back.
7. **Build documents in Python, not in the LLM.** The model supplies values; your code supplies structure.
8. **Human reviewed before merge.** Always.

---

## Tomorrow — Day 07: Architecture Diagram Generator

Build an agent that reads a PRD, searches past architecture decisions, extracts components,
and outputs a draw.io XML file and Terraform stubs — waiting for approval before saving.

---

## Reference Links

| Resource | URL |
|----------|-----|
| LangGraph docs | https://langchain-ai.github.io/langgraph/ |
| `ToolNode` / `tools_condition` | https://langchain-ai.github.io/langgraph/reference/prebuilt/ |
| ChatOllama | https://python.langchain.com/docs/integrations/chat/ollama/ |
| ChromaDB docs | https://docs.trychroma.com |
| nomic-embed-text | https://ollama.com/library/nomic-embed-text |
