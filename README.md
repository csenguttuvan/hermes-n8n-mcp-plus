# Hermes n8n MCP Plus


Local stdio MCP bridge for managing n8n from Hermes Agent — now with write tools.


This is a fork of `CyberSamuraiX/hermes-n8n-mcp`, extended with `create_workflow`, `update_workflow`, and `delete_workflow` on top of the original read-only/ops tool set. It gives Hermes full n8n workflow management without exposing n8n over the public internet and without putting API keys in your Hermes config.


## What it does


Exposes these MCP tools:


- `health` — check n8n API reachability and optional Docker container status
- `list_workflows` — list workflows, optionally filtered by active state
- `get_workflow` — inspect one workflow with secret-bearing fields redacted
- `find_workflows` — search workflow metadata
- `list_executions` — list recent executions
- `get_execution` — inspect one execution; payload data is off by default
- `recent_failures` — recent failed/error executions
- `export_workflow` — fetch redacted workflow JSON for backup/review
- `activate_workflow` — activate a workflow by ID
- `deactivate_workflow` — deactivate a workflow by ID
- `container_logs` — optional Docker logs with line-level redaction
- `create_workflow` — create a new workflow from a JSON definition. Dry-run by default.
- `update_workflow` — patch an existing workflow by ID (name, nodes, connections, settings, tags). Previews current state before applying. Dry-run by default.
- `delete_workflow` — permanently delete a workflow by ID. Previews an export backup before deleting. Dry-run by default.


All three write tools require an explicit `confirm=true` argument to actually mutate anything. Called with `confirm=false` (the default), they return a preview of what would happen and make no API call that changes n8n.


## Security posture


- Stdio only. No HTTP server. No public port.
- API key is loaded from environment or a local dotenv file.
- `.env` is gitignored.
- Example config uses `REPLACE_ME`, never a real key.
- Tool responses redact obvious credential, token, secret, password, and authorization fields.
- Execution payload data is disabled by default in `get_execution`.
- Workflow activation/deactivation, create, update, and delete are all production mutations. Treat them like loaded weapons.
- Write tools default to a dry-run preview; nothing is created, patched, or deleted unless the caller explicitly passes `confirm=true`.
- `delete_workflow` always fetches an export/backup preview of the workflow before a confirmed delete goes through.


## Requirements


- Python 3.10+
- Hermes Agent with native MCP enabled
- n8n API key
- n8n reachable from the machine running Hermes, usually `http://127.0.0.1:5678`


### Critical dependency pin: mcp==1.29.0


The official `mcp` PyPI package released a breaking v2.0.0 on 2026-07-28 that removed `mcp.server.fastmcp` entirely (renamed to `MCPServer`, moved module paths, swapped `httpx` for `httpx2`, and more). If `requirements.txt` uses a loose constraint like `mcp>=1.29.0`, `pip install` will resolve to 2.0.0 and the server will crash on import with:


```text
ModuleNotFoundError: No module named 'mcp.server.fastmcp'
```


This repo pins `mcp==1.29.0` (the last stable pre-v2 release) as a hard pin, not a floor. Do not loosen this constraint until the codebase is migrated to the v2 `MCPServer` API. If you ever see the above error, check `pip show mcp` — if it reports 2.0.0 or newer, run:


```bash
pip uninstall -y mcp
pip install "mcp==1.29.0"
```


## Install


```bash
git clone https://github.com/csenguttuvan/hermes-n8n-mcp-plus.git
cd hermes-n8n-mcp-plus
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
```


Verify the install landed on the correct SDK version before going further:


```bash
pip show mcp
python -c "from mcp.server.fastmcp import FastMCP; print('OK')"
```


## Store your n8n key


Manual version:


```bash
install -d -m 700 ~/.config/n8n-mcp-plus
cat > ~/.config/n8n-mcp-plus/env <<'EOF'
N8N_BASE_URL=http://127.0.0.1:5678
N8N_API_KEY=REPLACE_ME
N8N_MCP_TIMEOUT=30
N8N_CONTAINER_NAME=n8n
N8N_MCP_ALLOW_DOCKER_LOGS=true
EOF
chmod 600 ~/.config/n8n-mcp-plus/env
```


Replace `REPLACE_ME` locally. Do not commit the real file.


This fork's tools also read `N8N_API_KEY` / `N8N_API_URL` directly from the environment, not only from a dotenv file, so you can alternatively inject them straight from `~/.hermes/config.yaml` using `${N8N_API_KEY}` interpolation — see below.


## Hermes config


Add this to `~/.hermes/config.yaml` under `mcp_servers`. If the original `n8n` bridge is already registered, add this as a second, separate entry (`n8n_plus`) rather than replacing it — this keeps a safe read-only fallback available:


```yaml
mcp_servers:
  n8n:
    command: /Users/admin/.hermes/mcp-installs/n8n/.venv/bin/python
    args:
      - /Users/admin/.hermes/mcp-installs/n8n/server.py
    enabled: true
    env:
      N8N_API_KEY: "${N8N_API_KEY}"
      N8N_API_URL: "http://localhost:5678/api/v1"

  n8n_plus:
    command: /Users/admin/projects/hermes-n8n-mcp-plus/.venv/bin/python
    args:
      - /Users/admin/projects/hermes-n8n-mcp-plus/server.py
    enabled: true
    env:
      N8N_API_KEY: "${N8N_API_KEY}"
      N8N_API_URL: "http://localhost:5678/api/v1"
```


Indentation matters. Both `n8n:` and `n8n_plus:` must sit at the same indent level, directly under `mcp_servers:`, with no other top-level key breaking the block in between. Validate the file parses correctly before reloading:


```bash
python3 -c "import yaml; d = yaml.safe_load(open('/Users/admin/.hermes/config.yaml')); print(list(d.get('mcp_servers', {}).keys()))"
```


Then reload MCP in Hermes:


```text
/reload-mcp
```


Or from shell:


```bash
hermes mcp test n8n_plus
```


Tools register with the server-name prefix, e.g. `mcp__n8n_plus__create_workflow`, `mcp__n8n_plus__health`, distinct from the original bridge's `mcp__n8n__*` tools if running both side by side.


## Smoke test outside Hermes


```bash
. .venv/bin/activate
python -m py_compile server.py
python -c "import server; print('imported OK')"
hermes mcp test n8n_plus
```


If `import server` hangs or throws `ModuleNotFoundError: No module named 'mcp.server.fastmcp'`, re-check the `mcp==1.29.0` pin above — this is almost always a dependency version problem, not a code problem.


## Using the write tools


All three write tools follow the same dry-run-by-default pattern. Example flow for `create_workflow`:


```text
Call mcp__n8n_plus__create_workflow with workflow={"name": "test", "nodes": [], "connections": {}} and confirm=false.
```


Returns a preview, no mutation:


```json
{
  "ok": false,
  "error": "Dry run only. Set confirm=true to create the workflow.",
  "workflow_preview": { "name": "test", "nodes": [], "connections": {} }
}
```


Once the preview looks right, re-run with `confirm=true` to actually create it. The same pattern applies to `update_workflow` (previews current state and proposed patch) and `delete_workflow` (previews an export backup before deleting).


Recommended test order for any new environment: `create_workflow` then `list_workflows` to confirm it landed, then `update_workflow`, then `delete_workflow` — each with a throwaway workflow, verified against the n8n UI at each step.


## Docker logs


`container_logs` shells out to Docker. If the user running Hermes cannot access Docker, set:


```text
N8N_MCP_ALLOW_DOCKER_LOGS=false
```


The rest of the API tools will still work.


## Notes for production use


- Keep n8n bound to loopback behind your reverse proxy.
- Do not expose this MCP bridge over Caddy, nginx, or Docker ports.
- Rotate n8n API keys if they ever hit chat logs, terminals, CI output, screenshots, or issue trackers.
- Back up workflows before mutating them. `update_workflow` and `delete_workflow` both preview state before confirming, but always check the preview yourself before passing `confirm=true`.
- Never loosen the `mcp==1.29.0` pin in `requirements.txt` without first testing against the `MCPServer` v2 API.


## Roadmap


- Migrate from `FastMCP` (v1.x) to `MCPServer` (v2.x) once the v2 API stabilizes and this fork's tool set is verified compatible.
- Consider adding `run_workflow` as a fourth write tool for manual execution triggers.


## License


MIT. See `LICENSE`.
