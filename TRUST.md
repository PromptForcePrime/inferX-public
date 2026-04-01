# InferX — Privacy and Data Handling

## The short version

InferX runs on your machine. Your prompts go directly from your machine to Anthropic or OpenAI.
InferX intercepts them locally to inject caching headers and route models. It does not log
prompt content, does not send data to any third-party service, and does not have any
telemetry enabled by default.

## What InferX does with your data

| Data | Where it goes | How long |
|------|--------------|----------|
| API requests (prompts) | Forwarded to Anthropic/OpenAI unchanged | Not stored |
| Prefix hashes | Local Redis, for cache detection | TTL from config (default 24h) |
| Session turn count | Local Redis, for context trimming | TTL from config (default 2h) |
| Cost/token metadata | Local SQLite (`~/.inferx/data/inferx.db`) | Forever (your data) |
| API keys | Environment variables only, never written to disk | — |

**Prompt content is never written to disk or logged.** The SQLite database stores only
token counts, model names, timestamps, and cost figures — not the content of requests.

## Network traffic

InferX makes outbound connections to exactly two destinations:

1. `api.anthropic.com` — your Anthropic API calls
2. `api.openai.com` — your OpenAI API calls (if configured)

No other outbound connections are made. There is no analytics endpoint, no error reporting
service, no update check, no usage telemetry.

### Verify it yourself

```bash
# Watch all outbound TLS traffic while making an API call.
# You should see only api.anthropic.com (or api.openai.com).
sudo tcpdump -i any -n 'dst port 443' 2>/dev/null
```

```bash
# Or use lsof to see what connections InferX has open:
lsof -i -n -P | grep inferx
```

## The RouteLLM sidecar

The router sidecar (`promptforce/inferx-sidecar`) runs entirely locally. It uses a
pre-trained matrix-factorization model included in the Docker image. It does not make any
network calls. It receives only the prompt text (locally, over Docker networking) and returns
a routing decision (`strong` or `weak`).

## Fail-open guarantee

If any InferX component fails — Redis is down, the sidecar times out, anything errors —
the proxy forwards the original request to the upstream API unchanged. InferX never
blocks an API call because of its own failure.

This is a hard design constraint. The proxy is instrumented to log internal errors but will
never return a 5xx to the caller due to its own state. The upstream's response is always
returned as-is.

## What we can't see

We (PromptForcePrime) have zero visibility into your usage. InferX is a local binary. We
do not operate any servers that your requests touch. We cannot see your prompts, your API
keys, your usage volume, or your cost data.

## Data you can delete

Everything InferX writes is in `~/.inferx/`:

```
~/.inferx/
├── data/
│   ├── inferx.db        ← SQLite: cost/token logs (no prompt content)
│   └── config.yaml      ← your configuration
├── inferx.log           ← proxy logs (no prompt content)
└── inferx.pid           ← PID file for binary mode
```

Delete `~/.inferx/` to remove all InferX data. Uninstall by removing the binary and
reverting `~/.claude/settings.json`.

## Source availability

The proxy binary is distributed in obfuscated form (garble + -ldflags="-s -w"). We do this
to protect proprietary routing logic, not to hide data handling. The data handling described
in this document is verifiable with `tcpdump` and `lsof` regardless of binary obfuscation.

The RouteLLM sidecar (`sidecar/`) uses the open-source RouteLLM library (Apache 2.0).
The router service code is available in the public repo.

## Contact

Security disclosures: see [SECURITY.md](./SECURITY.md)
Other questions: open an issue in this repo
