# molecule-ai-workspace-template-deepagents

A Molecule AI workspace template for the **deepagents** multi-agent orchestration runtime. It provisions a pre-configured Docker environment in which one root orchestrator agent and one or more subordinate task agents operate concurrently, coordinating via shared executor, escalation, and provider plumbing.

## Purpose

`deepagents` is designed for workflows that require parallel task agents with a single orchestration entry point. The template provides:

- A `config.yaml` that describes agent roles, skill lists, and timeouts
- An `adapter.py` that translates Molecule platform events to the deepagents runtime and vice versa
- An `executor.py` that runs sub-agent tasks, enforcing timeout and cancellation semantics
- An `escalation.py` that propagates task failures, circuit-break triggers, and human-in-the-loop requests
- A `providers.py` that routes model selection across available LLM endpoints
- A `system-prompt.md` that seeds the orchestrator's system prompt with multi-agent conventions
- A `requirements.txt` pinning runtime dependencies
- A `Dockerfile` that bundles everything into a runnable image

## Key Files and Their Roles

| File | Role |
|------|------|
| `config.yaml` | Declarative workspace config: schema version, agent roles, skill requirements per role, default timeouts, escalation thresholds |
| `adapter.py` | Translates Molecule platform API events into in-process function calls understood by the orchestrator and executor |
| `executor.py` | Manages the sub-agent lifecycle: spawn, track progress, enforce per-task timeout, propagate results |
| `escalation.py` | Receives failure signals from the executor and decides whether to retry, delegate to another agent, or surface an error to the platform |
| `providers.py` | Routes model selection per task type; reads `MODEL_ROUTING` env var to override the default model for a given role |
| `system-prompt.md` | Base system prompt injected into every new orchestrator session; defines multi-agent conventions, tool namespaces, and escalation protocol |
| `requirements.txt` | Runtime Python dependencies |
| `Dockerfile` | Multi-stage build: installs deps, copies workspace files, sets entrypoint |

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `MOLECULE_PLATFORM_URL` | Yes | — | Base URL of the Molecule platform API |
| `MOLECULE_WORKSPACE_ID` | Yes | — | Workspace instance identifier |
| `MOLECULE_AGENT_ROLE` | No | `orchestrator` | Role this container fulfills (`orchestrator` or `task`) |
| `MOLECULE_API_KEY` | Yes | — | API key for platform authentication |
| `DEEPAGENTS_MODEL_PRIMARY` | No | `claude-sonnet-4-20250514` | Primary model for orchestrator |
| `DEEPAGENTS_MODEL_TASK` | No | `claude-haiku-4-20250514` | Default model for sub-agent tasks |
| `DEEPAGENTS_TASK_TIMEOUT_SEC` | No | `300` | Default timeout (seconds) for sub-agent task execution |
| `DEEPAGENTS_MAX_RETRIES` | No | `2` | Number of automatic retries before escalation |
| `DEEPAGENTS_CIRCUIT_BREAK_THRESHOLD` | No | `5` | Consecutive failures before circuit break is triggered |
| `DEEPAGENTS_ESCALATION_WEBHOOK_URL` | No | — | Webhook called when a task escalates |
| `MODEL_ROUTING` | No | — | JSON map of `{"role": "model-id"}` overriding provider routing |
| `MOLECULE_SKILLS_DIR` | No | `/workspace/skills` | Directory containing per-skill prompt fragments |

## Skill Loading

Skills are loaded at startup by `adapter.py` calling into `providers.py`'s skill bootstrap. The directory referenced by `MOLECULE_SKILLS_DIR` is scanned for `.md` files; each is prepended to the system prompt of every agent role listed in `config.yaml`'s `skills` block for that role.

Skill files are not bundled into the Docker image at build time — they are mounted as a volume at runtime so operators can hot-reload prompts without rebuilding the image.

## Multi-Agent Conventions

The orchestrator and task agents communicate through structured messages written to a shared work directory (`/workspace/agent-shared/`). File-based communication is used because the runtime is designed to run both the orchestrator and task agents in the same container, with each sub-agent process writing its output to a predictable path:

```
/workspace/agent-shared/
  orchestrator/
    intent.yaml          # Top-level task intent from platform
    plan/
      step-001.yaml      # Ordered task decomposition
  task-agents/
    task-001/
      input.yaml         # Task parameters
      output.yaml        # Task result
      status.yaml        # pending | running | done | failed | escalated
    task-002/
      ...
```

Task agents are spawned as subprocesses by `executor.py`. The orchestrator monitors `status.yaml` files using a polling loop (configurable interval via `DEEPAGENTS_STATUS_POLL_INTERVAL_SEC`, default 5 s).

## Development Setup

```bash
# 1. Clone the template
git clone https://github.com/your-org/molecule-ai-workspace-template-deepagents.git
cd molecule-ai-workspace-template-deepagents

# 2. Create a virtual environment
python -m venv .venv
source .venv/bin/activate    # Windows: .venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Set required env vars (create a .env file or export)
export MOLECULE_PLATFORM_URL=https://platform.molecule.ai
export MOLECULE_WORKSPACE_ID=ws-dev-local
export MOLECULE_API_KEY=your-dev-key-here
export DEEPAGENTS_MODEL_PRIMARY=claude-sonnet-4-20250514
export DEEPAGENTS_MODEL_TASK=claude-haiku-4-20250514

# 5. Build the Docker image locally
docker build -t deepagents-local:latest .

# 6. Run a smoke test
docker run --rm \
  --env-file .env \
  -v "$(pwd)/skills:/workspace/skills:ro" \
  deepagents-local:latest python -m executor_smoke_test
```

## Testing

### Unit tests

```bash
pytest tests/unit/ -v
```

### Integration tests

Integration tests require a running Molecule platform and valid credentials.

```bash
export MOLECULE_PLATFORM_URL=https://platform.molecule.ai
export MOLECULE_WORKSPACE_ID=ws-integration-test
export MOLECULE_API_KEY=your-integration-key
export DEEPAGENTS_TASK_TIMEOUT_SEC=60
pytest tests/integration/ -v
```

### Smoke test (no platform required)

```bash
python -m executor_smoke_test
```

This runs `executor.py` against a mock task queue and verifies that status files are written correctly and that timeouts are enforced.

## Release Process

1. **Version bump** — Update the version in `__init__.py` (`__version__`) and tag the commit: `git tag -a v0.x.y -m "Release v0.x.y"`.
2. **CI pipeline** — Push the tag to trigger CI: lint (`ruff`), type check (`mypy`), unit tests, Docker build and push to the container registry.
3. **Changelog** — Add entries to `CHANGELOG.md` under the new version heading.
4. **Notify** — Update the Molecule platform workspace templates registry by opening a PR against `molecule-ai/workspace-registry` with the new tag and updated SHA.

## Repository Structure

```
molecule-ai-workspace-template-deepagents/
├── Dockerfile
├── config.yaml
├── requirements.txt
├── adapter.py
├── __init__.py
├── escalation.py
├── executor.py
├── providers.py
├── system-prompt.md
├── skills/               # optional; mounted at runtime
├── CLAUDE.md
├── known-issues.md
└── runbooks/
    └── local-dev-setup.md
```
