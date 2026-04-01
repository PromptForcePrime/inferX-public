# InferX — Design Philosophy

## The problem we're solving

LLM API costs are dominated by two things: repeated tokens and over-specified models.

Every time you send a request with a long system prompt, you pay for those tokens again —
even if you sent the exact same system prompt 10 seconds ago. Every time you ask a simple
factual question, you pay Sonnet prices even though Haiku would answer it correctly.

These are not API design flaws. They are limitations that can be addressed at the proxy
layer, transparently, without modifying application code.

## Three principles

**1. One line of config.**
InferX must be adoptable with a single `base_url` change. If it requires application
refactoring, it's solving the wrong problem. The value should be immediately visible on the
first request with no learning curve.

**2. Fail open, always.**
InferX runs between your code and your API provider. If it fails, your code must still work.
Every internal error results in a passthrough — never a 5xx returned to the caller. This is
not a nice-to-have. It is the only design that is acceptable for infrastructure that sits in
the critical path.

**3. No data leaves your machine.**
InferX operates entirely on localhost. It does not log prompts, does not report usage, does
not phone home. Your API keys and conversation content are yours. The binary is obfuscated
to protect proprietary routing logic, not to obscure data handling — which you can verify
independently with tcpdump.

## What we chose not to build

**We did not build a managed service.** A managed proxy would reduce the infrastructure
burden on users but would require trusting a third party with API keys and prompt content.
That tradeoff is not acceptable for the use cases InferX targets (developer tooling, internal
agents, production workloads). Local-first is not a positioning decision — it is the
only architecture consistent with principle 3.

**We did not build a UI-first product.** The dashboard is a debugging and monitoring tool,
not the product. The product is a URL. Developer tools should have lean interfaces that
stay out of the way.

**We did not make caching opt-in.** Prefix caching is on by default because it has no
downside: cached tokens cost less and the cache is local. Making it opt-in would reduce
adoption and leave money on the table for no reason.

## On model routing

Routing is the most opinionated of the three layers. The decision of when to use a
cheaper model is not always obvious, and a wrong routing decision has a cost: either
you pay for a model you didn't need (under-routing) or you get a weaker answer
than the task required (over-routing).

InferX uses RouteLLM's matrix-factorization router as the default because it has the
best published benchmark performance with the lowest inference overhead. The threshold
is configurable. Shadow mode lets you observe routing decisions without acting on them,
so you can calibrate the threshold for your workload before going live.

## The cost math

The three layers target different token categories:

- **Prefix cache**: Reduces repeated input tokens (system prompts, tool definitions)
- **Session memory**: Reduces conversation history re-transmission
- **Model routing**: Reduces per-token price by using a cheaper model where appropriate

A Claude Code session is a good example. The system prompt is long (~4k tokens) and
identical across most requests. Session memory re-sends prior turns on every request.
With InferX: the system prompt is cached (90% cost reduction on those tokens), only
the new turn is sent (not the full history), and simple clarification questions route
to Haiku. The compound effect is 40–70% of the total bill.

## Open questions

Things we're still figuring out:

- How to handle model routing when users have strong model preferences
- Whether session memory should be opt-in for privacy-sensitive workloads
- How to extend InferX to other providers (Vertex, Bedrock, Groq)
- Whether a shared baseline dataset would let us publish routing benchmarks

If you have opinions on any of these, open an issue.
