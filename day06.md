# Day 06 — Agentic CI/CD Build

## Story
> "Devraj's pipeline failed at 2am. The agent read the log, found the past fix, drafted the PR, and waited for his approval — before his phone rang."

**Duration:** 1.5 hours  
**Local model:** `llama3.2:3b` + `nomic-embed-text`  
**New installs:** `snap install drawio`  
**Prereqs:** Day 05 `ci_knowledge_base` on disk at `~/day5/ci_knowledge_base`

---

## The Problem

The Day 05 RAG system can find similar failures. That's retrieval.

But retrieval alone doesn't fix anything. You still need someone to:
- Read the log
- Search the knowledge base
- Form a root cause
- Write a PR description
- Get a human to approve it before anything changes

At 2am, that someone is Devraj. Every time.

---

## The Solution — After Day 06

Same CI failure. Same knowledge base. But now an agent drives the entire workflow:

```
1. read_log_file("ci_failure_gha.log")
   → 1233 bytes of npm ERESOLVE errors

2. search_knowledge_base("npm ERESOLVE peer dependency conflict react version")
   → Match 1: 79.4% similar — nodejs / dependency_conflict
     Fix: add --legacy-peer-deps or upgrade the shared package

3. analyze_root_cause(log + past failures context)
   → { "error_type": "dependency_conflict", "root_cause": "...",
       "fix_command": "npm install --legacy-peer-deps",
       "severity": "high", "confidence": "high" }

4. draft_pr_description(analysis + past failures)
   → "## CI Failure — Automated Root Cause Analysis\n..."

5. post_to_slack(draft, "#ci-alerts")
   → [PAUSED] "Post this to Slack? [Y/n]:"
   → ✅ Approved — posted
```

---

## Concept 1: Tool Ordering — Enforced, Not Trusted

Small local models like `llama3.2:3b` do not reliably follow a system prompt ordering instruction. If you rely on the prompt alone, the model may call `post_to_slack` first with a placeholder.

This agent enforces order in code with three mechanisms:

**`REQUIRED_SEQUENCE`** — the canonical order:
```python
REQUIRED_SEQUENCE = [
    "read_log_file",
    "search_knowledge_base",
    "analyze_root_cause",
    "draft_pr_description",
    "post_to_slack",
]
```

**`NEXT_STEP_GUIDANCE`** — injected into the conversation after each tool completes:
```python
NEXT_STEP_GUIDANCE = {
    "read_log_file": "Good. Now call search_knowledge_base with query='npm ERESOLVE...'",
    "search_knowledge_base": "Good. Now call analyze_root_cause with log_content and past_failures.",
    ...
}
```

**`FORCE_PROMPTS`** — used when the model returns no tool call at all:
```python
FORCE_PROMPTS = {
    "read_log_file": "Call read_log_file with filepath='ci_failure_gha.log'.",
    ...
}
```

---

## Concept 2: `tool_state` — Passing Data Between Tools

The model often passes placeholders (`<PR description>`) or empty strings as arguments. `tool_state` is a dict that accumulates the actual outputs from each tool. `_fill_args` then injects the real value before the next tool is called:

```python
tool_state = {}

# After read_log_file:
tool_state["log_content"] = result

# After search_knowledge_base:
tool_state["past_failures"] = result

# After analyze_root_cause:
tool_state["analysis_json"] = result

# After draft_pr_description:
tool_state["pr_description"] = result
```

When `post_to_slack` is called, `_fill_args` always replaces `message` with `tool_state["pr_description"]` — the actual PR description, not whatever placeholder the model passed.

---

## Concept 3: The Manual Agentic Loop

This agent does not use LangGraph. It runs a manual `while True` loop:

```
                    ┌──────────────────────┐
                    │      START           │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │   call_ollama()      │  ← sends messages + TOOL_DEFINITIONS
                    └──────────┬───────────┘
                               │
               ┌───────────────▼────────────────┐
               │   tool_calls in response?       │
               └───────────────┬────────────────┘
                     YES       │        NO
              ┌────────────────┘        └────────────────────┐
   ┌──────────▼──────────┐                    ┌──────────────▼──────────┐
   │  _fill_args()       │                    │  next required tool?     │
   │  execute tool       │                    │  YES → inject FORCE_PROMPT│
   │  save to tool_state │                    │  NO  → break (done)      │
   │  inject NEXT_STEP   │                    └─────────────────────────┘
   └──────────┬──────────┘
              │
         loop back
```

The loop exits when all 5 tools in `REQUIRED_SEQUENCE` appear in `tools_called`.

---

## Concept 4: Human-in-the-Loop Guardrail

`post_to_slack` does not post automatically. It shows the full draft and waits for `Y`:

```
======================================================================
📣 PROPOSED SLACK POST → #ci-alerts
======================================================================
## CI Failure — Automated Root Cause Analysis
...
======================================================================
Post this to Slack? [Y/n]:
```

Any tool that sends data outside localhost gets this gate. The agent cannot bypass it.

---

## Setup — Before Labs Start

```bash
mkdir -p ~/day6 && cd ~/day6

ollama list | grep llama3.2
ollama list | grep nomic-embed
python3 -c "import chromadb; print('chromadb ok')"
```

Verify your Day 05 knowledge base is on disk:

```bash
ls ~/day5/ci_knowledge_base/
```

**Expected:** `chroma.sqlite3`

If missing:

```bash
cd ~/day5 && python3 embed_failures.py && cd ~/day6
```

---

## Lab 1 — CI Failure Analyzer: Local (llama3.2:3b) ⏱️ 30 min

**Goal:** Build the 5-tool CI failure analyzer. The agent reads a failure log, searches the Day 05 knowledge base, analyzes the root cause, drafts a PR description, and posts to Slack for approval — in guaranteed order.

The agent is split across two files:

| File | What it contains |
|------|-----------------|
| `lab1_tools.py` | Configuration, knowledge base setup, and the 5 tool functions |
| `lab1_ci_agent_local.py` | Tool registry, TOOL_DEFINITIONS, ordering logic, and the agentic loop |

---

### Step 1.1 — Create the failure log

```bash
cat > ~/day6/ci_failure_gha.log << 'EOF'
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
EOF
```

---

### Step 1.2 — Create `lab1_tools.py` (configuration + 5 tool functions)

See `~/day6/lab1_tools.py` — already on disk. Contains:
- `MODEL`, `EMBED_MODEL`, URLs, `LOG_FILE`, `KB_DIR`
- `SAMPLE_FAILURES` + `_ensure_knowledge_base()` (seeds ChromaDB if empty)
- Tools 1–5: `read_log_file`, `search_knowledge_base`, `analyze_root_cause`, `draft_pr_description`, `post_to_slack`

---

### Step 1.3 — Create `lab1_ci_agent_local.py` (tool registry + agentic loop)

```bash
cat > ~/day6/lab1_ci_agent_local.py << 'EOF'
import sys
import os
import json
import requests
import chromadb

# ============================================================================
# CONFIGURATION
# ============================================================================

SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))
MODEL       = "llama3.2:3b"
EMBED_MODEL = "nomic-embed-text"
OLLAMA_CHAT_URL     = "http://localhost:11434/api/chat"
OLLAMA_GENERATE_URL = "http://localhost:11434/api/generate"
OLLAMA_EMBED_URL    = "http://localhost:11434/api/embeddings"
LOG_FILE = "ci_failure_gha.log"
KB_DIR   = os.path.expanduser("~/day5/ci_knowledge_base")

print("=" * 70)
print("Day 06 — CI Failure Analyzer  (5-tool agentic loop)")
print(f"Model  : {MODEL}")
print(f"Log    : {LOG_FILE}")
print("=" * 70)
print()

# ============================================================================
# KNOWLEDGE BASE  — seed once, then query
# ============================================================================

SAMPLE_FAILURES = [
    {
        "id": "kf001",
        "text": (
            "npm install failed with ERESOLVE peer dependency conflict. "
            "React version mismatch: project requires react@18 but a shared package "
            "requires react@17. Fixed by adding --legacy-peer-deps flag or upgrading "
            "the shared package to support react@18."
        ),
        "metadata": {"error_type": "dependency_conflict", "tool": "npm"},
    },
    {
        "id": "kf002",
        "text": (
            "Docker build failed: COPY failed, file not found. "
            "A required artifact was not generated in a prior build step. "
            "Fixed by ensuring the build step runs before the Docker COPY."
        ),
        "metadata": {"error_type": "missing_artifact", "tool": "docker"},
    },
    {
        "id": "kf003",
        "text": (
            "pytest failed with ImportError: cannot import name 'X' from 'Y'. "
            "A module was renamed or removed. Fixed by updating the import path."
        ),
        "metadata": {"error_type": "import_error", "tool": "pytest"},
    },
    {
        "id": "kf004",
        "text": (
            "GitHub Actions OOMKilled: process ran out of memory. "
            "Build required more than 7GB RAM. Fixed by splitting the build or "
            "upgrading the runner to a larger instance type."
        ),
        "metadata": {"error_type": "oom", "tool": "github_actions"},
    },
    {
        "id": "kf005",
        "text": (
            "eslint failed with 'Parsing error: The keyword const is reserved'. "
            "Node.js version mismatch — the CI runner was using Node 10 while "
            "the code requires Node 16+. Fixed by pinning node-version in the workflow."
        ),
        "metadata": {"error_type": "node_version", "tool": "eslint"},
    },
]


def _embed(text: str) -> list:
    resp = requests.post(
        OLLAMA_EMBED_URL,
        json={"model": EMBED_MODEL, "prompt": text},
        timeout=30,
    )
    resp.raise_for_status()
    return resp.json()["embedding"]


def _ensure_knowledge_base() -> chromadb.Collection:
    client = chromadb.PersistentClient(path=KB_DIR)
    col = client.get_or_create_collection(
        "ci_failures",
        metadata={"hnsw:space": "cosine"},
    )
    if col.count() == 0:
        print("  📚 Seeding knowledge base with sample failures...")
        embeddings = [_embed(f["text"]) for f in SAMPLE_FAILURES]
        col.add(
            ids=[f["id"] for f in SAMPLE_FAILURES],
            embeddings=embeddings,
            documents=[f["text"] for f in SAMPLE_FAILURES],
            metadatas=[f["metadata"] for f in SAMPLE_FAILURES],
        )
        print(f"     ✅ {col.count()} documents indexed")
    else:
        print(f"  📚 Knowledge base ready ({col.count()} documents)")
    return col


KB_COLLECTION = _ensure_knowledge_base()
print()


# ============================================================================
# TOOL 1 — READ LOG FILE
# ============================================================================

def read_log_file(filepath: str = LOG_FILE) -> str:
    resolved = (
        filepath if os.path.isabs(filepath)
        else os.path.join(SCRIPT_DIR, filepath)
    )
    print(f"  📂 read_log_file: {resolved}")
    try:
        with open(resolved) as f:
            content = f.read()
        print(f"     ✅ {len(content)} bytes read")
        return content
    except FileNotFoundError:
        return f"Error: file not found at {resolved}"
    except Exception as e:
        return f"Error reading file: {e}"


# ============================================================================
# TOOL 2 — SEARCH KNOWLEDGE BASE
# ============================================================================

def search_knowledge_base(query: str, n_results: int = 2) -> str:
    print(f"  🔎 search_knowledge_base: '{query[:60]}'")
    try:
        query_embedding = _embed(query)
        results = KB_COLLECTION.query(
            query_embeddings=[query_embedding],
            n_results=n_results,
        )
        docs      = results.get("documents", [[]])[0]
        metas     = results.get("metadatas", [[]])[0]
        distances = results.get("distances", [[]])[0]

        if not docs:
            return "No similar failures found in knowledge base."

        output_parts = []
        for i, (doc, meta, dist) in enumerate(zip(docs, metas, distances), 1):
            similarity = round(max(0, (1 - dist)) * 100, 1)
            output_parts.append(
                f"[Match {i} — {similarity}% similar | type={meta.get('error_type')}]\n{doc}"
            )

        output = "\n\n".join(output_parts)
        print(f"     ✅ {len(docs)} match(es) found")
        print()
        print("  ─── ① RAG ran BEFORE analysis ✓ ───────────────────────────")
        print("  ─── ② Similarity Scores ────────────────────────────────────")
        for i, (doc, meta, dist) in enumerate(zip(docs, metas, distances), 1):
            similarity = round(max(0, (1 - dist)) * 100, 1)
            print(f"     📌 Match {i}: {similarity}% similar")
            print(f"        Category : {meta.get('category', meta.get('error_type', 'N/A'))}")
            print(f"        History  : {doc[:120]}...")
            print()
        return output

    except Exception as e:
        return f"Error searching knowledge base: {e}"


# ============================================================================
# TOOL 3 — ANALYZE ROOT CAUSE
# ============================================================================

def analyze_root_cause(log_content: str, past_failures: str = "") -> str:
    print()
    print("  🔍 analyze_root_cause: calling model...")

    context = (
        f"\nSimilar past failures from knowledge base:\n{past_failures}"
        if past_failures else ""
    )

    prompt = f"""You are a DevOps engineer analyzing a CI pipeline failure.
Analyze the CI log below. Return ONLY valid JSON with ALL six fields filled in.

Required JSON structure (fill every field, never leave a field empty or "Unknown"):
{{
  "error_type": "dependency_conflict",
  "root_cause": "One sentence: npm ERESOLVE because @company/shared-utils@1.2.3 requires react@^17.0.0 but project installs react@18.2.0",
  "affected_file": "package.json",
  "fix_command": "npm install --legacy-peer-deps",
  "severity": "high",
  "confidence": "high"
}}

Rules:
- error_type: a short snake_case label for the category of failure
- root_cause: one factual sentence based on the log only
- affected_file: the file or config that must change (e.g. package.json, workflow.yml)
- fix_command: the exact shell command or config change that resolves the issue
- severity: low, medium, or high based on whether the pipeline is completely blocked
- confidence: low, medium, or high — how certain you are given the log clarity and KB matches
- Return only raw JSON, no markdown fences, no explanation.

CURRENT CI FAILURE LOG:
{log_content}
{context}
"""

    response = requests.post(
        OLLAMA_GENERATE_URL,
        json={{"model": MODEL, "prompt": prompt, "format": "json", "stream": False}},
        timeout=120,
    )
    response.raise_for_status()
    result = response.json()["response"]

    print("     ✅ Analysis complete")
    try:
        parsed = json.loads(result)
        print()
        print("  ─── ④ JSON Completeness Check ──────────────────────────────")
        for field in ["error_type", "root_cause", "fix_command", "severity", "confidence"]:
            val = parsed.get(field)
            status = "✅" if val and val.lower() not in ("unknown", "") else "❌ MISSING"
            print(f"     {status} {field}: {str(val)[:80]}")
    except json.JSONDecodeError:
        print("     ⚠️  Response was not valid JSON")

    return result


# ============================================================================
# TOOL 4 — DRAFT PR DESCRIPTION
# ============================================================================

def draft_pr_description(analysis_json: str, past_failures: str = "") -> str:
    print()
    print("  📝 draft_pr_description: generating markdown...")

    try:
        analysis = json.loads(analysis_json)
    except json.JSONDecodeError:
        analysis = {{"root_cause": analysis_json}}

    similar_note = ""
    if past_failures:
        lines = past_failures.split("\n")
        first_doc_snippet = ""
        for line in lines[1:]:
            if line.strip():
                first_doc_snippet = line.strip()[:120]
                break
        similar_note = (
            f"\n> This issue is similar to a previous CI failure: "
            f"_{first_doc_snippet}_\n"
        )

    history_section = (
        f"\n### Historical Context\n{similar_note}\n{past_failures[:600]}"
        if past_failures else ""
    )

    pr_body = f"""## CI Failure — Automated Root Cause Analysis

### Error
**Type:** {analysis.get('error_type', 'Unknown')}
**Severity:** {analysis.get('severity', 'Unknown')}
**Confidence:** {analysis.get('confidence', 'Unknown')}

### Root Cause
{analysis.get('root_cause', 'Not determined')}

### Affected File / Config
`{analysis.get('affected_file', 'Unknown')}`

### Proposed Fix
```bash
{analysis.get('fix_command', '# No fix command generated')}
```
{history_section}
---
*Generated by CI Failure Analyzer Agent — human reviewed before merge*"""

    print(f"     ✅ PR description ready ({len(pr_body)} chars)")
    print()
    print("  ─── ③ Historical Failure Used in PR? ──────────────────────────")
    if similar_note:
        print(f"  — {similar_note.strip()[:120]}")
    else:
        print("     ❌ No KB history was available")
    return pr_body


# ============================================================================
# TOOL 5 — POST TO SLACK  (human-in-the-loop)
# ============================================================================

def post_to_slack(message: str, channel: str = "#ci-alerts") -> str:
    display = message.replace("\\n", "\n").replace("\\t", "\t")

    print()
    print("=" * 70)
    print(f"📣 PROPOSED SLACK POST → {channel}")
    print("=" * 70)
    print(display[:800] + ("..." if len(display) > 800 else ""))
    print("=" * 70)

    try:
        approval = input("Post this to Slack? [Y/n]: ").strip().lower()
    except EOFError:
        approval = "n"

    if approval in ("", "y", "yes"):
        print(f"  ✅ Approved — posted to {channel}")
        return f"Posted to {channel}. Human approved."

    print("  ❌ Rejected — not posted")
    return "Rejected by engineer. Not posted."


# ============================================================================
# TOOL REGISTRY
# ============================================================================

TOOL_FUNCS = {{
    "read_log_file":         read_log_file,
    "search_knowledge_base": search_knowledge_base,
    "analyze_root_cause":    analyze_root_cause,
    "draft_pr_description":  draft_pr_description,
    "post_to_slack":         post_to_slack,
}}

TOOL_DEFINITIONS = [
    {{
        "type": "function",
        "function": {{
            "name": "read_log_file",
            "description": "Read a CI failure log file from disk and return its raw content.",
            "parameters": {{
                "type": "object",
                "properties": {{"filepath": {{"type": "string", "description": "Path to the CI failure log file."}}}},
                "required": ["filepath"],
            }},
        }},
    }},
    {{
        "type": "function",
        "function": {{
            "name": "search_knowledge_base",
            "description": "Search the CI failure knowledge base for historically similar failures. Call right after read_log_file.",
            "parameters": {{
                "type": "object",
                "properties": {{"query": {{"type": "string", "description": "Short description of the current CI error."}}}},
                "required": ["query"],
            }},
        }},
    }},
    {{
        "type": "function",
        "function": {{
            "name": "analyze_root_cause",
            "description": "Analyze the CI failure root cause. Pass the raw log and past similar failures from the KB search.",
            "parameters": {{
                "type": "object",
                "properties": {{
                    "log_content":   {{"type": "string", "description": "Raw CI log from read_log_file."}},
                    "past_failures": {{"type": "string", "description": "Similar past failures from search_knowledge_base."}},
                }},
                "required": ["log_content"],
            }},
        }},
    }},
    {{
        "type": "function",
        "function": {{
            "name": "draft_pr_description",
            "description": "Create a GitHub PR description in Markdown from the root cause analysis.",
            "parameters": {{
                "type": "object",
                "properties": {{
                    "analysis_json": {{"type": "string", "description": "JSON string from analyze_root_cause."}},
                    "past_failures": {{"type": "string", "description": "Similar past failures for historical context."}},
                }},
                "required": ["analysis_json"],
            }},
        }},
    }},
    {{
        "type": "function",
        "function": {{
            "name": "post_to_slack",
            "description": "Post the PR description to Slack for engineer review. Final step — requires human approval.",
            "parameters": {{
                "type": "object",
                "properties": {{
                    "message": {{"type": "string", "description": "The full PR description to post."}},
                    "channel": {{"type": "string", "description": "Slack channel name."}},
                }},
                "required": ["message"],
            }},
        }},
    }},
]

# ============================================================================
# REQUIRED TOOL SEQUENCE
# ============================================================================

REQUIRED_SEQUENCE = [
    "read_log_file",
    "search_knowledge_base",
    "analyze_root_cause",
    "draft_pr_description",
    "post_to_slack",
]

NEXT_STEP_GUIDANCE = {{
    "read_log_file": (
        "Good. You have the CI log. "
        "Now call search_knowledge_base with query='npm ERESOLVE peer dependency conflict react version'."
    ),
    "search_knowledge_base": (
        "Good. You have the knowledge base results. "
        "Now call analyze_root_cause with log_content=<the full log> and past_failures=<the KB result>."
    ),
    "analyze_root_cause": (
        "Good. You have the root cause analysis JSON. "
        "Now call draft_pr_description with analysis_json=<the JSON from analyze_root_cause>."
    ),
    "draft_pr_description": (
        "Good. You have the PR description. "
        "Now call post_to_slack with message=<the full PR description markdown text>."
    ),
}}

FORCE_PROMPTS = {{
    "read_log_file":         f"Call read_log_file with filepath='{LOG_FILE}'.",
    "search_knowledge_base": "Call search_knowledge_base with query='npm ERESOLVE peer dependency conflict'.",
    "analyze_root_cause":    "Call analyze_root_cause with log_content from the log file and past_failures from search.",
    "draft_pr_description":  "Call draft_pr_description with analysis_json from the root cause analysis.",
    "post_to_slack":         "Call post_to_slack with message containing the full PR description.",
}}

# ============================================================================
# OLLAMA API CALL
# ============================================================================

def _trim_messages(messages: list) -> list:
    trimmed = []
    for msg in messages:
        if msg.get("role") == "tool":
            content = msg.get("content", "")
            if len(content) > 1500:
                content = content[:1500] + "\n...[truncated]"
            trimmed.append({{**msg, "content": content}})
        else:
            trimmed.append(msg)
    return trimmed


def call_ollama(messages: list) -> dict:
    try:
        response = requests.post(
            OLLAMA_CHAT_URL,
            json={{
                "model":    MODEL,
                "messages": _trim_messages(messages),
                "tools":    TOOL_DEFINITIONS,
                "stream":   False,
            }},
            timeout=180,
        )
        response.raise_for_status()
        return response.json()["message"]
    except requests.exceptions.ConnectionError:
        print("\n❌ Cannot reach Ollama. Run:  ollama serve")
        sys.exit(1)
    except requests.exceptions.Timeout:
        print("❌ Ollama timed out.")
        sys.exit(1)


# ============================================================================
# AGENTIC LOOP
# ============================================================================

ARG_ALIASES = {{
    "read_log_file": {{
        "filepath": ["filepath", "file_path", "path", "filename", "file", "log_path", "log_file"],
    }},
    "search_knowledge_base": {{
        "query": ["query", "failure_string", "error_string", "search_query",
                  "failure_description", "description", "error", "log_lines",
                  "log_content", "content", "text", "search_term"],
    }},
    "analyze_root_cause": {{
        "log_content":   ["log_content", "log_string", "log", "content", "ci_log",
                          "log_data", "failure_log", "log_lines", "raw_log", "log_text",
                          "ci_failure_log", "log_file_content"],
        "past_failures": ["past_failures", "similar_failures", "kb_results",
                          "historical_context", "context", "rag_results",
                          "knowledge_base_results", "search_results"],
    }},
    "draft_pr_description": {{
        "analysis_json": ["analysis_json", "analysis", "root_cause_json", "root_cause",
                          "analysis_result", "json_analysis", "root_cause_analysis",
                          "analysis_data", "pr_body", "pr_content", "json_result"],
        "past_failures": ["past_failures", "similar_failures", "context",
                          "historical_context", "kb_results"],
    }},
    "post_to_slack": {{
        "message": ["message", "pr_description", "content", "text", "body",
                    "pr_body", "slack_message", "description", "markdown",
                    "pr_markdown", "pr_text"],
        "channel": ["channel", "slack_channel"],
    }},
}}

PLACEHOLDER_PATTERNS = {{"<PrDescription>", "<pr_description>", "<message>",
                          "<PR description>", "PR description here"}}


def _normalize_args(name: str, args: dict) -> dict:
    aliases = ARG_ALIASES.get(name, {{}})
    normalized = {{}}
    all_alias_keys = {{v for variants in aliases.values() for v in variants}}
    for canonical, variants in aliases.items():
        for variant in variants:
            if variant in args:
                normalized[canonical] = args[variant]
                break
    for k, v in args.items():
        if k not in all_alias_keys:
            normalized[k] = v
    return normalized if normalized else args


def _is_placeholder(value: str) -> bool:
    return any(p in value for p in PLACEHOLDER_PATTERNS) or (
        value.startswith("<") and value.endswith(">")
    )


def _fill_args(name: str, args: dict, tool_state: dict) -> dict:
    args = _normalize_args(name, args)

    if name == "analyze_root_cause":
        if "log_content" not in args or not args.get("log_content"):
            if "log_content" in tool_state:
                args["log_content"] = tool_state["log_content"]
        if "past_failures" not in args and "past_failures" in tool_state:
            args["past_failures"] = tool_state["past_failures"]

    elif name == "draft_pr_description":
        if "analysis_json" in tool_state:
            candidate = str(args.get("analysis_json", ""))
            try:
                json.loads(candidate)
            except (json.JSONDecodeError, ValueError):
                args["analysis_json"] = tool_state["analysis_json"]
        if not args.get("analysis_json") or _is_placeholder(str(args.get("analysis_json", ""))):
            if "analysis_json" in tool_state:
                args["analysis_json"] = tool_state["analysis_json"]
        # Always inject past_failures from tool_state — model often passes wrong value
        if "past_failures" in tool_state:
            args["past_failures"] = tool_state["past_failures"]

    elif name == "post_to_slack":
        if "pr_description" in tool_state:
            args["message"] = tool_state["pr_description"]
        else:
            msg = args.get("message", "")
            if not msg or _is_placeholder(str(msg)):
                fallback = tool_state.get("analysis_json", "")
                if fallback:
                    args["message"] = fallback

    return args


def run_agent():
    messages = [
        {{
            "role": "system",
            "content": (
                "You are a CI failure analyzer. "
                "Call tools one at a time. "
                "Follow user instructions for which tool to call next."
            ),
        }},
        {{
            "role": "user",
            "content": f"Step 1: Call read_log_file with filepath='{LOG_FILE}'.",
        }},
    ]

    step = 0
    tool_state = {{}}
    tools_called = []

    def _next_required_tool():
        for tool in REQUIRED_SEQUENCE:
            if tool not in tools_called:
                return tool
        return None

    while True:
        step += 1
        print(f"\n--- Step {{step}} ---")

        message = call_ollama(messages)
        messages.append(message)
        tool_calls = message.get("tool_calls", [])

        if not tool_calls:
            next_tool = _next_required_tool()
            if next_tool is None:
                print()
                print("=" * 70)
                print("✅ AGENT COMPLETED — all 5 tools called")
                print("=" * 70)
                content = message.get("content", "")
                print(content[:2000] if content else "(workflow complete)")
                print("=" * 70)
                break
            print(f"  ⚡ No tool call — forcing: {{next_tool}}")
            messages.append({{"role": "user", "content": FORCE_PROMPTS[next_tool]}})
            continue

        last_successful_tool = None
        for tool_call in tool_calls:
            fn   = tool_call["function"]
            name = fn["name"]
            args = fn.get("arguments", {{}})

            if isinstance(args, str):
                try:
                    args = json.loads(args)
                except json.JSONDecodeError:
                    args = {{}}

            args = _fill_args(name, args, tool_state)
            print(f"  🔧 Tool call: {{name}}")

            if name not in TOOL_FUNCS:
                result = f"Error: unknown tool '{{name}}'"
                print(f"  ❌ {{result}}")
                messages.append({{"role": "tool", "content": result}})
                continue

            try:
                result = TOOL_FUNCS[name](**args)

                if name == "read_log_file":
                    tool_state["log_content"] = result
                elif name == "search_knowledge_base":
                    tool_state["past_failures"] = result
                elif name == "analyze_root_cause":
                    tool_state["analysis_json"] = result
                elif name == "draft_pr_description":
                    tool_state["pr_description"] = result

                if name not in tools_called:
                    tools_called.append(name)
                last_successful_tool = name

            except TypeError as e:
                result = f"Error calling {{name}}: {{e}}"
                print(f"  ❌ {{result}}")

            messages.append({{"role": "tool", "content": str(result)}})

        if "post_to_slack" in tools_called:
            print()
            print("=" * 70)
            print("✅ AGENT COMPLETED — all 5 tools called")
            print("=" * 70)
            print(f"Tools called in order: {{tools_called}}")
            print("=" * 70)
            break

        if last_successful_tool and last_successful_tool in NEXT_STEP_GUIDANCE:
            next_tool = _next_required_tool()
            if next_tool:
                messages.append({{
                    "role": "user",
                    "content": NEXT_STEP_GUIDANCE[last_successful_tool],
                }})


try:
    run_agent()
except KeyboardInterrupt:
    print("\n⚠️  Interrupted by user.")
    sys.exit(1)
EOF
```

---

### Step 1.4 — Run Lab 1

```bash
cd ~/day6 && python3 lab1_ci_agent_local.py
```

Type `Y` when the approval prompt appears.

### Expected output

```
======================================================================
Day 06 — CI Failure Analyzer  (5-tool agentic loop)
Model  : llama3.2:3b
Log    : ci_failure_gha.log
======================================================================

  📚 Knowledge base ready (5 documents)

--- Step 1 ---
  🔧 Tool call: read_log_file
  📂 read_log_file: .../ci_failure_gha.log
     ✅ 1233 bytes read

--- Step 2 ---
  🔧 Tool call: search_knowledge_base
  🔎 search_knowledge_base: 'npm ERESOLVE peer dependency conflict react version'
     ✅ 2 match(es) found

  ─── ① RAG ran BEFORE analysis ✓ ───────────────────────────
  ─── ② Similarity Scores ────────────────────────────────────
     📌 Match 1: 85.3% similar
        Category : dependency_conflict
        History  : npm install failed with ERESOLVE peer dependency conflict...

--- Step 3 ---
  🔧 Tool call: analyze_root_cause
  🔍 analyze_root_cause: calling model...
     ✅ Analysis complete

  ─── ④ JSON Completeness Check ──────────────────────────────
     ✅ error_type: dependency_conflict
     ✅ root_cause: npm ERESOLVE because @company/shared-utils...
     ✅ fix_command: npm install --legacy-peer-deps
     ✅ severity: high
     ✅ confidence: high

--- Step 4 ---
  🔧 Tool call: draft_pr_description
  📝 draft_pr_description: generating markdown...
     ✅ PR description ready (1227 chars)

  ─── ③ Historical Failure Used in PR? ──────────────────────────
  — > This issue is similar to a previous CI failure: _Node.js build failed..._

--- Step 5 ---
  🔧 Tool call: post_to_slack

======================================================================
📣 PROPOSED SLACK POST → #ci-alerts
======================================================================
## CI Failure — Automated Root Cause Analysis
...
======================================================================
Post this to Slack? [Y/n]: Y
  ✅ Approved — posted to #ci-alerts

======================================================================
✅ AGENT COMPLETED — all 5 tools called
======================================================================
Tools called in order: ['read_log_file', 'search_knowledge_base',
                        'analyze_root_cause', 'draft_pr_description',
                        'post_to_slack']
======================================================================
```

---

## Lab 2 (Bonus) — Break the Tool Order

### Why Day 4 worked without enforcement but Day 6 doesn't

In Day 4 (`lab2_agent_ollama.py`) there were only 2 tools and no data chaining — the model wrote its own summary after reading the log. The system prompt was a simple numbered list and the model could hold the full plan in one shot.

Day 6 has 5 tools with heavy chaining: `log_content` from tool 1 must reach tool 3, `past_failures` from tool 2 must reach both tool 3 and tool 4. With `llama3.2:3b`, after a tool returns, the model often responds with text ("I found the error...") instead of calling the next tool. The longer the chain, the more often it forgets which step it's on or passes placeholder args like `<log content>` instead of real data.

| | Day 4 | Day 6 |
|---|---|---|
| Tools | 2 | 5 |
| Data chaining | None | Heavy (each tool feeds next) |
| Works with system prompt only | ✅ | ❌ |
| Needs `NEXT_STEP_GUIDANCE` + `FORCE_PROMPTS` | No | Yes |

**The rule:** for simple 2-step flows, a good system prompt is enough. Once you have chained data passing across 3+ tools with a small local model, you need code-level enforcement.

### Run Lab 2

The agent has a built-in Lab 2 mode — no manual commenting needed. Open `lab1_ci_agent_local.py` and change one line near the top:

```python
LAB2_MODE = True   # was False
```

Run it:

```bash
python3 lab1_ci_agent_local.py
```

### What you will observe

```
⚠️  LAB 2 MODE — enforcement disabled (no NEXT_STEP_GUIDANCE, no FORCE_PROMPTS)

--- Step 1 ---
  🔧 Tool call: read_log_file   ← works (initial user message forces it)

--- Step 2 ---
  ⚠️  Model returned text, not a tool call (attempt 1)
      Expected: search_knowledge_base
      Model said: "The CI log shows an npm ERESOLVE error..."

--- Step 3 ---
  ⚠️  Model returned text, not a tool call (attempt 2)
      ...

❌ LAB 2 RESULT — agent gave up after 10 steps
   Tools completed : ['read_log_file']
   Tools missing   : ['search_knowledge_base', 'analyze_root_cause', 'draft_pr_description', 'post_to_slack']

   WHY: without NEXT_STEP_GUIDANCE and FORCE_PROMPTS the model
   returns text instead of tool calls — the pipeline never advances.
```

| Without enforcement | With enforcement |
|---------------------|-----------------|
| Model returns text after first tool | Model calls next tool immediately |
| Pipeline stalls — only 1 of 5 tools run | All 5 tools called in correct order |
| `past_failures` never reaches `analyze_root_cause` | KB results passed correctly |
| Agent aborts at step 10 | Agent completes in ~6 steps |

### How Lab 1 recovers when the model returns text

When the model returns text instead of a tool call, Lab 1 has two recovery layers built into the loop:

**Attempt 1 & 2 — `FORCE_PROMPTS` injected as a new user message:**
The code detects no tool call and appends an explicit instruction directly into the conversation:
```
⚡ No tool call — forcing: search_knowledge_base (attempt 1)
```
The model sees a new user message: *"Call search_knowledge_base with query='npm ERESOLVE...'"* and responds with the tool call.

**Attempt 3 — context reset:**
If the model still doesn't respond after 2 forced prompts, the growing back-and-forth (text responses + force prompts) has become too large and is confusing it. The code strips the conversation down to just `[system message] + [last tool result] + [fresh force prompt]` — a clean context that the model can act on.
```
🔄 Context reset — clearing history to unblock search_knowledge_base
```

In Lab 2 both layers are disabled — text responses accumulate, the model never gets redirected, and it hits the step limit.

### What this teaches

- A 2-tool pipeline can rely on system prompt alone — a 5-tool chain cannot
- Small local models (`llama3.2:3b`) talk about what they should do instead of doing it
- `NEXT_STEP_GUIDANCE` and `FORCE_PROMPTS` are the enforcement layer — the model is the executor, not the orchestrator
- In production agents, workflow correctness depends on what the **code** guarantees, not what the model remembers

**Restore before continuing:** set `LAB2_MODE = False` in `lab1_ci_agent_local.py`.

---

## Lab 3 (Bonus) — Add Today's Failure to the Knowledge Base

Add to `SAMPLE_FAILURES` in `lab1_ci_agent_local.py`:

```python
{
    "id": "kf006",
    "text": (
        "GitHub Actions npm install failed: ERESOLVE unable to resolve dependency tree. "
        "react@18.2.0 in root conflicts with peer dependency react@^17.0.0 from "
        "@company/shared-utils@1.2.3. Fix: upgrade @company/shared-utils to a version "
        "compatible with react@18, or add --legacy-peer-deps to npm install."
    ),
    "metadata": {"error_type": "dependency_conflict", "tool": "npm"},
},
```

Delete only the `ci_failures` collection so it re-seeds with the new entry (keeps other Day 05 data):

```bash
python3 -c "
import chromadb, os
client = chromadb.PersistentClient(path=os.path.expanduser('~/day5/ci_knowledge_base'))
client.delete_collection('ci_failures')
print('Collection cleared — will re-seed on next run')
"
python3 lab1_ci_agent_local.py
```

The similarity score for Match 1 should jump above 90%.

---

## Day 06 Recap — What You Built Today

| Lab | What you did |
|-----|-------------|
| Lab 1 | 5-tool CI agent — manual agentic loop, forced tool ordering, `tool_state` |
| Lab 2 (Bonus) | Broke ordering enforcement to observe the failure mode |
| Lab 3 (Bonus) | Added today's failure to the knowledge base |

### File layout

```
~/day6/
├── ci_failure_gha.log          ← mock GitHub Actions failure
├── lab1_tools.py               ← config + knowledge base + 5 tool functions
└── lab1_ci_agent_local.py      ← tool registry + ordering logic + agentic loop

~/day5/
└── ci_knowledge_base/
    └── chroma.sqlite3          ← ChromaDB on disk (loaded at startup)
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
| `read_log_file` | `filepath: str` | raw log content |
| `search_knowledge_base` | `query: str` | similarity matches from ChromaDB |
| `analyze_root_cause` | `log_content, past_failures` | JSON: error_type, root_cause, affected_file, fix_command, severity, confidence |
| `draft_pr_description` | `analysis_json, past_failures` | Markdown PR body with historical context |
| `post_to_slack` | `message, channel` | approval status string |

### Golden Rules

1. **RAG before analysis** — `search_knowledge_base` must run before `analyze_root_cause`
2. **Enforce order in code** — small local models ignore system prompt ordering; use `REQUIRED_SEQUENCE` + `NEXT_STEP_GUIDANCE` + `FORCE_PROMPTS`
3. **`tool_state` carries real values** — never trust the model to pass arguments correctly; `_fill_args` injects from `tool_state`
4. **Guardrail external actions** — `post_to_slack` always requires `Y`
5. **`"format": "json"`** — always set this when calling llama3.2:3b for structured output

---

## Tomorrow — Day 07: Architecture Diagram Generator

Build an agent that reads a PRD, searches past architecture decisions, extracts components, and outputs a draw.io XML file and Terraform stubs — waiting for approval before saving.

---

## Reference Links

| Resource | URL |
|----------|-----|
| ChromaDB docs | https://docs.trychroma.com |
| nomic-embed-text | https://ollama.com/library/nomic-embed-text |
