# agy-cli — Antigravity CLI Integration Tools

This repository provides tools and wrappers to integrate the official **Google Antigravity CLI** (`agy`) with an active **Antigravity Pro subscription** (Google One AI Premium OAuth credentials) into OpenAI-compatible clients, agent frameworks, and developer environments.

No AI Studio API keys or developer registrations are needed; all requests are authenticated natively through your local OAuth token using the official CLI under the hood.

---

## Features

1. **Transparent TTY Wrapper (`agy_wrapper.py`)**:
   Prevents background/non-interactive calls (e.g. from agent frameworks) from hanging when the CLI expects standard input. Automatically redirects `stdin` to `/dev/null` in non-interactive sessions while preserving 100% of the interactive TUI.
2. **OpenAI-Compatible API Proxy (`agy_api.py`)**:
   Exposes a local FastAPI/Uvicorn server hosting the standard `/v1/chat/completions` endpoint. It translates chat completion requests into official `agy` command executions — including streaming, thinking/reasoning extraction (`reasoning_content`), model mapping, per-session conversation persistence, and retries with real failure-reason surfacing (quota exhaustion, expired login) from per-run logs.
3. **FastMCP Server (`agy_mcp.py`)**:
   Exposes the `ask_antigravity` tool to other agents via Model Context Protocol (MCP).
4. **Daemon/Service templates**:
   `agy-daemon.sh` (remote-control daemon setup for Linux/macOS) and `agy-api.service` (systemd unit for the API proxy) for persistent background operation.

---

## Installation & Setup

### 1. Install Dependencies
Clone this repository and set up a virtual environment:
```bash
uv venv
source .venv/bin/activate
uv pip install -e .
```

### 2. Standalone Wrapper Setup
Rename your original `agy` binary to `agy.original` and link this wrapper script to your path:
```bash
mv ~/.local/bin/agy ~/.local/bin/agy.original
cp agy_wrapper.py ~/.local/bin/agy
chmod +x ~/.local/bin/agy
```

### 3. API Proxy Background Service (Linux)
Place the proxy in a dedicated directory and set up a clean virtualenv:
```bash
mkdir -p ~/.hermes/agy-api
cp agy_api.py ~/.hermes/agy-api/
uv venv ~/.hermes/agy-api/.venv
uv pip install fastapi uvicorn --python ~/.hermes/agy-api/.venv/bin/python
```

Adapt `agy-api.service` to your paths and install it as a systemd unit:
```bash
cp agy-api.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now agy-api
```

The proxy binds to `0.0.0.0:8000` by default. If you expose it beyond localhost, set an API token (see below).

For macOS, run the API via a LaunchAgent instead (see the upstream README history for a plist template).

### 4. Optional: API Authentication
Set `AGY_API_TOKEN` in the environment (e.g. `~/.hermes/.env`):
```bash
AGY_API_TOKEN=your-secret-token
```
When set, all `/v1/*` requests require `Authorization: Bearer <token>`; requests without it get a `401`. When unset, the proxy is open — fine for isolated lab networks, not for shared networks.

---

## Usage

### OpenAI API Proxy
Test the completions server locally:
```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "claude-sonnet-4.6", "messages": [{"role": "user", "content": "Say Hello"}]}'
```

Available model slugs (see `map_model_name` in `agy_api.py`): `gemini-3.8-flash`, `gemini-3.7-flash`, `gemini-3.6-flash`, `gemini-3.1-pro`, `claude-sonnet-4.6`, `claude-opus-4.6`, `gpt-oss-120b`.

### OpenAI-Compatible Client Config
Point any OpenAI-compatible client (e.g. Hermes Agent, Open WebUI) at the proxy:
```yaml
model: claude-sonnet-4.6
provider: custom
base_url: http://<proxy-host>:8000/v1
api_key: <token if AGY_API_TOKEN is set, otherwise any string>
```

### Session Persistence
Pass a `session_id` field in chat completion requests to reuse the same agy conversation across turns (multi-turn context). The proxy maps session IDs to agy conversation IDs automatically.

---

## Notes on this fork

- Linux-first paths (`~/.hermes/...` instead of macOS home directories)
- Slug-style model IDs (no spaces) so strict model pickers accept them
- Gemini 3.6/3.7/3.8 Flash model mappings
- Optional bearer-token auth (`AGY_API_TOKEN`)