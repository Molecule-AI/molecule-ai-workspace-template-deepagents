# Local Development Setup — deepagents

This runbook covers cloning, installing dependencies, building the Docker image, and running a local development loop for the deepagents workspace template.

## Prerequisites

- Python 3.11+
- Docker 24.0+
- `git`
- A Molecule platform account with API key and workspace ID

## 1. Clone the Repository

```bash
git clone https://github.com/your-org/molecule-ai-workspace-template-deepagents.git
cd molecule-ai-workspace-template-deepagents
```

If you are working on a fork, add upstream:

```bash
git remote add upstream https://github.com/your-org/molecule-ai-workspace-template-deepagents.git
```

## 2. Install Dependencies

Create a virtual environment and install Python packages:

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt
```

For linting and type checking (used in CI):

```bash
pip install ruff mypy pytest pytest-asyncio
```

## 3. Docker Build Test

Build the image and verify it starts without errors:

```bash
docker build -t deepagents-local:latest .
docker run --rm deepagents-local:latest --version
# Expected: prints the version from __init__.py
```

To rebuild without cached layers (prevents stale adapter.py / config.yaml issues):

```bash
docker build --no-cache -t deepagents-local:latest .
```

## 4. Configure Environment for Development

Create a `.env` file in the repo root (do not commit this file):

```
MOLECULE_PLATFORM_URL=https://platform.molecule.ai
MOLECULE_WORKSPACE_ID=ws-dev-local
MOLECULE_API_KEY=your-dev-key-here
MOLECULE_AGENT_ROLE=orchestrator
DEEPAGENTS_MODEL_PRIMARY=claude-sonnet-4-20250514
DEEPAGENTS_MODEL_TASK=claude-haiku-4-20250514
DEEPAGENTS_TASK_TIMEOUT_SEC=60
DEEPAGENTS_MAX_RETRIES=2
DEEPAGENTS_STATUS_POLL_INTERVAL_SEC=2
MOLECULE_SKILLS_DIR=/workspace/skills
```

You can override any variable at runtime without changing the file:

```bash
docker run --rm \
  --env-file .env \
  -e DEEPAGENTS_TASK_TIMEOUT_SEC=30 \
  deepagents-local:latest python -m executor
```

### Dev-only Overrides

| Variable | Dev default | Production default | Purpose |
|----------|-------------|-------------------|---------|
| `DEEPAGENTS_TASK_TIMEOUT_SEC` | `60` | `300` | Shorter timeout so dev loops are faster |
| `DEEPAGENTS_STATUS_POLL_INTERVAL_SEC` | `2` | `5` | Faster polling for local testing |
| `DEEPAGENTS_CIRCUIT_BREAK_THRESHOLD` | `2` | `5` | Easier to trigger circuit break during development |

## 5. Test the Executor Locally

The executor includes a built-in smoke test that runs without a platform connection:

```bash
# Activate venv
source .venv/bin/activate

# Run the smoke test
python -m executor_smoke_test
# Exit code 0 = pass; non-zero = failure
```

To run the full adapter integration test against a mock platform server:

```bash
# Start mock server in background
python -m mock_platform_server &
MOCK_PORT=$!

# Run integration tests
export MOLECULE_PLATFORM_URL=http://localhost:${MOCK_PORT}
export MOLECULE_WORKSPACE_ID=ws-test
export MOLECULE_API_KEY=test-key
pytest tests/integration/test_adapter.py -v
```

## 6. Common Issues

| Symptom | Likely cause | Resolution |
|---------|--------------|------------|
| `SchemaVersionMismatch` on startup | Stale Docker layer with old `adapter.py` | `docker build --no-cache`; pin `molecule-deepagents` version in `requirements.txt` |
| Tasks hang as `running` forever after timeout | `adapter.py` not forwarding timeout to platform | Set `DEEPAGENTS_ABORT_ON_TIMEOUT=1` or apply the fix in known-issues.md Issue 1 |
| `None`-valued model error on task agent | `MODEL_ROUTING` missing a role key | Always include all roles in the env var JSON; see known-issues.md Issue 2 |
| Infinite re-escalation loop | Retry counter not incremented on re-queue | Set `DEEPAGENTS_MAX_RETRIES=1` and `DEEPAGENTS_CIRCUIT_BREAK_THRESHOLD=1` as interim workaround |
| Skill prompts not loaded at runtime | `MOLECULE_SKILLS_DIR` not mounted as volume | Mount skills directory: `-v "$(pwd)/skills:/workspace/skills:ro"` |
| `docker build` fails with `pip install` error | Python version mismatch or outdated pip | Use Python 3.11+; `pip install --upgrade pip` before installing requirements |
| Permission denied on `/workspace/agent-shared` | Docker container running as non-root | Ensure the image is built with `USER appuser` or run with `--user $(id -u):$(id -g)` |

## 7. Hot-Reloading Skills

Skill prompt files are loaded from `MOLECULE_SKILLS_DIR` at runtime and do not require a Docker rebuild. To hot-reload:

```bash
# Edit a skill file
vim skills/my-skill.md

# The container picks up changes on the next task poll cycle (within DEEPAGENTS_STATUS_POLL_INTERVAL_SEC)
# No restart needed
```

For immediate reload during development, send `SIGHUP` to the running container process:

```bash
docker kill --signal HUP <container_id>
```
