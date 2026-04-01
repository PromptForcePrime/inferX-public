# InferX

**One-line change. 40–70% lower LLM costs.**

InferX is a local inference proxy that sits between your code and the Anthropic / OpenAI APIs. It adds prompt-caching, session memory, and intelligent model routing — invisibly.

```python
# Before
client = Anthropic()

# After — nothing else changes
client = Anthropic(base_url="http://localhost:8019")
```

For Claude Code, it is a single JSON edit:

```json
// ~/.claude/settings.json
{"env": {"ANTHROPIC_BASE_URL": "http://localhost:8019"}}
```

---

## Install

**One-line install (Mac + Linux):**

```bash
curl -fsSL https://install.inferx.dev | bash
```

The installer detects whether Docker is available and picks the right mode automatically.

**Start:**

```bash
./start.sh
```

**Dashboard:**

```
http://localhost:8019/dashboard
```

---

## How it works

Every request passes through three layers, each one reducing cost:

```
Request → [1] Prefix cache  → [2] Session memory → [3] Model router → Anthropic/OpenAI
```

| Layer | What it does | Typical saving |
|-------|-------------|----------------|
| **Prefix cache** | Detects repeated system prompts; injects `anthropic-beta: prompt-caching` so Anthropic charges 10× less for cached tokens | 20–40% |
| **Session memory** | Stores prior turns in Redis; trims redundant context sent back on each turn | 10–20% |
| **Model router** | Routes simple requests to Haiku / GPT-4o-mini; sends complex ones to Sonnet / GPT-4o | 15–30% |

If Redis is down, the sidecar times out, or anything internal errors — the request forwards to the upstream API unchanged. **InferX never blocks your work.**

---

## Three modes

InferX starts in `baseline` mode and advances automatically. Use `inferx report` to see your actual vs. projected costs at any point.

| Mode | What's active | When to use |
|------|--------------|-------------|
| `baseline` | Proxy only — logs "would have" projections, no optimisations applied | Days 1–7 |
| `shadow` | Prompt caching active; routing and session off | Days 8–14 |
| `live` | All three layers active | Day 15+ |

Change mode in `config.yaml`:

```yaml
inferx:
  mode: live   # baseline | shadow | live
```

---

## Configuration

InferX works without a config file — open `http://localhost:8019` on first start for the guided setup.

For file-based config, copy the example and fill in your keys:

```bash
cp config.example.yaml config.yaml
```

Key options:

```yaml
inferx:
  customer_id: acme           # your identifier (shows in dashboard)
  mode: live

  upstream:
    anthropic_api_key: ${ANTHROPIC_API_KEY}   # or set env var
    openai_api_key: ${OPENAI_API_KEY}

  router:
    enabled: true
    strong_model: claude-sonnet-4-6
    weak_model: claude-haiku-4-5-20251001
    cost_threshold: 0.3        # 0 = always weak · 1 = always strong

  kv_cache:
    enabled: true
    ttl_seconds: 86400         # 24 h
    min_prefix_tokens: 100     # minimum tokens to qualify for caching

  session_store:
    enabled: true
    ttl_seconds: 7200          # 2 h
    max_turns: 20
```

Full schema: [`config.example.yaml`](config.example.yaml)

---

## CLI

```bash
# Start the proxy (auto-detects Docker vs binary)
./start.sh

# Force modes
./start.sh --docker
./start.sh --binary

# Print version
inferx --version

# Generate a cost report from the last 30 days
inferx report

# Custom date range
inferx report --days 7

# Custom database path
inferx report --db /path/to/inferx.db
```

---

## Dashboard

Live metrics at `http://localhost:8019/dashboard`:

- Total cost vs. projected (what you would have paid without InferX)
- Savings breakdown: routing, caching, session
- Request log with per-request decisions
- Cache hit rate and session hit rate
- Model distribution

---

## Docker (full stack)

```bash
# Start all services (proxy + router sidecar + Redis)
docker compose up -d

# Check health
curl localhost:8019/health

# View logs
docker compose logs -f proxy

# Stop
docker compose down
```

The `docker-compose.yml` does not expose Redis (`:6379`) or the router sidecar (`:8001`) outside the Docker network. Only the proxy port (`8019`) is reachable from your host.

---

## Dev builds

```bash
# Local dev build (no garble, fast)
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build

# Plain Go binary
go build -o inferx ./cmd/inferx

# Run tests
go test ./... -race

# Release build (obfuscated + stripped, matches CI)
garble -literals -tiny build -ldflags="-s -w" -o inferx ./cmd/inferx

# Verify binary-only image (must show only: inferx)
docker run --rm promptforce/inferx:latest ls /app
```

---

## Privacy

InferX **never logs prompt or response text**. It logs token counts, cost, latency, and routing decisions — the same metadata your API provider already has.

Verify with `tcpdump` or any packet inspector that no prompt data leaves your machine to InferX infrastructure:

```bash
# macOS — watch for any outbound connections that are NOT api.anthropic.com
sudo tcpdump -i any 'tcp and not host api.anthropic.com' -A 2>/dev/null | grep -v '^$'
```

Full data-handling details: [`TRUST.md`](TRUST.md)

---

## Integration examples

**Python (Anthropic SDK):**
```python
import anthropic
client = anthropic.Anthropic(base_url="http://localhost:8019")
```

**Python (OpenAI SDK → Anthropic):**
```python
from openai import OpenAI
client = OpenAI(api_key="sk-ant-...", base_url="http://localhost:8019/v1")
```

**Node.js:**
```js
import Anthropic from "@anthropic-ai/sdk";
const client = new Anthropic({ baseURL: "http://localhost:8019" });
```

**Claude Code:**
```json
// ~/.claude/settings.json
{"env": {"ANTHROPIC_BASE_URL": "http://localhost:8019"}}
```

**cURL:**
```bash
curl http://localhost:8019/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-sonnet-4-6","max_tokens":256,"messages":[{"role":"user","content":"hi"}]}'
```

---

## Troubleshooting

**Proxy not starting**
```bash
# Check logs
cat ~/.promptforce/inferx.log          # binary mode
docker compose logs proxy          # Docker mode
```

**Claude Code not routing through InferX**
```bash
# Verify settings.json
cat ~/.claude/settings.json
# Should contain: "ANTHROPIC_BASE_URL": "http://localhost:8019"
# Restart Claude Code after editing settings.json
```

**No savings showing in dashboard**
- Run in `baseline` mode for at least a few hours to build a baseline.
- Check that `min_prefix_tokens` matches your typical system-prompt size.
- Confirm Redis is running: `docker compose ps redis`

**Router sidecar timeout (decision: strong)**
- This is expected when the sidecar is cold-starting. InferX fails open to the strong model.
- Warm up by sending a few requests, then check: `docker compose logs router-sidecar`

---

## License

See [LICENSE](LICENSE). The Go proxy binary is distributed as a compiled, obfuscated binary. Source for the sidecar (RouteLLM, Apache 2.0) is in [`sidecar/`](sidecar/).
