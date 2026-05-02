# template-deepagents

Molecule AI workspace template for the **deepagents** runtime.

## Usage

### In Molecule AI canvas
Select this template when creating a new workspace — it appears in the template picker automatically.

### From a URL (community install)
Paste this URL when creating a workspace:
```
github://Molecule-AI/template-deepagents
```

## Files
- `config.yaml` — workspace configuration (runtime, model, skills, etc.)
- `system-prompt.md` — agent system prompt (if present)

## Schema version
`template_schema_version: 1` — compatible with Molecule AI platform v1.x.

## License
Business Source License 1.1 — © Molecule AI.

## See also
For the multi-agent architecture (orchestrator + task agents, file-based coordination via `/workspace/agent-shared/`), the full `config.yaml` schema, environment variables, skill loading rules, dev setup, and release process, see [`CLAUDE.md`](CLAUDE.md).
