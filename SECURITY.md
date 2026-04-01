# Security Policy

## Supported versions

InferX is currently in open beta. Security fixes are applied to the latest release only.

| Version | Supported |
|---------|-----------|
| latest  | Yes       |
| older   | No        |

## Reporting a vulnerability

**Do not file a public GitHub issue for security vulnerabilities.**

Email: security@inferx.dev

Include:
- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Any suggested fixes (optional)

We will acknowledge receipt within 48 hours and aim to publish a fix within 14 days for
critical issues.

## Scope

In scope:
- The InferX proxy binary
- The RouteLLM router sidecar
- The install script (`install.inferx.dev`)
- The InferX website

Out of scope:
- Anthropic or OpenAI API behavior
- Redis or Docker vulnerabilities (report to upstream)
- Issues requiring physical access to the machine running InferX

## Design constraints relevant to security

**API keys** are read from environment variables only. They are never written to disk by InferX.

**Localhost only**: The proxy binds to `localhost:8000` by default. It is not intended to be
exposed to the internet. If you expose it, treat it as an authenticated service — InferX
does not implement its own auth layer (it passes through your API key).

**No outbound connections** except to `api.anthropic.com` and `api.openai.com`. The install
script fetches from `install.inferx.dev` (Vercel-hosted) and Docker Hub.

**Fail-open**: All internal failures result in a passthrough to the upstream API. This means
a compromised InferX component can at most observe traffic — it cannot block it.
