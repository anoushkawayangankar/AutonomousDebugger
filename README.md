# TryCatchMe

TryCatchMe is a local, browser-based workspace for running and repairing **Python** snippets. It combines a React code editor with a FastAPI service that executes pasted code, classifies its errors, attempts automated repairs, and returns both the revised code and a line-level change log.

It is designed as a local development tool: the browser app calls `http://localhost:8000`, and the repair engine can use a local Qwen model through `llama-cpp-python` or an Ollama installation. There is no hosted service, user account system, project persistence, or remote API integration in this repository.

## What you can do

- Write or paste Python into a Monaco editor, or upload a `.py` file.
- Run the code and read its stdout, stderr, error status, and elapsed time in the terminal panel.
- Describe the intended change and ask the repair engine to fix the code.
- Review added and removed lines generated during repair, then revert the editor to its pre-repair contents.
- Switch the interface between light and dark themes and resize the editor, output, and patch panels.

## How repair works

The Python engine runs a bounded repair loop. It executes code, parses common Python exceptions, checks selected logic patterns, applies string/AST-based cleanup, and uses a local language model when an LLM fix is selected. It produces a timestamped JSON report in `code-autofix-engine/iterations/` containing code snapshots, execution results, repair methods, and changes.

The engine includes heuristics for syntax and name/import issues plus a small set of logic patterns such as Fibonacci memoization, tree traversal order, and binary-search pointer or midpoint mistakes. These are heuristics, not a proof of correctness. Review every generated patch before using it in another project.

## Repository layout

| Path | Purpose |
| --- | --- |
| `frontend/` | Vite, React, TypeScript, Tailwind, shadcn/ui, and Monaco editor interface. |
| `code-autofix-engine/` | Python runner, repair loop, FastAPI server, local-model adapter, and report writer. |
| `code-autofix-engine/runtime/sandbox_runner.py` | Runs temporary Python files for the API; it also contains unused JavaScript and Java helpers. |
| `code-autofix-engine/config/settings.py` | Iteration limits and local model backend settings. |

## Run locally

Use two terminals from the repository root.

### 1. Start the Python API

```sh
cd code-autofix-engine
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt fastapi uvicorn
uvicorn api_server:app --reload --host 127.0.0.1 --port 8000
```

`fastapi` and `uvicorn` are required by `api_server.py`, but are not currently listed in `requirements.txt`, so the command installs them explicitly.

For LLM-based repairs, the default configuration expects a GGUF model named `qwen2.5-coder-7b-instruct-q4_k_m.gguf` at:

```text
code-autofix-engine/models/gguf/qwen2.5-coder-7b-instruct-q4_k_m.gguf
```

Alternatively, change `MODEL_BACKEND` and `MODEL_NAME` in `code-autofix-engine/config/settings.py` to use a locally available Ollama model. Without a working local model, runs and AST-based fixes can still work, but LLM-dependent repairs will not produce a meaningful correction.

### 2. Start the web app

```sh
cd frontend
npm ci
npm run dev
```

Vite serves the interface on `http://localhost:8080`. The API permits this origin through its CORS configuration.

## API

### `POST /run`

Executes the supplied Python code once and returns output plus a parsed error type.

```json
{ "code": "print('hello')" }
```

### `POST /repair`

Runs the repair loop and returns the final code, the saved report path, a summary of the final iteration, and line-level changes.

```json
{
  "code": "print(total / 0)",
  "prompt": "Handle a zero total safely.",
  "max_iterations": 5
}
```

## Current constraints

- The user-facing API and editor are Python-only, even though the low-level runner has JavaScript and Java helpers.
- The execution runner uses a temporary file and a timeout; it does **not** enforce the filesystem, memory, or network isolation claimed by the older component README. Do not treat it as a safe sandbox for untrusted code.
- Repair reports include submitted code snapshots and are written to disk locally.
- The frontend uses fixed localhost URLs, so it needs the API running on port 8000 unless the source is changed.

## Useful commands

```sh
# Frontend static checks and production bundle
cd frontend
npm run lint
npm run build

# Python engine's built-in sample (requires its configured LLM for a useful repair)
cd ../code-autofix-engine
python3 main.py --test
```
