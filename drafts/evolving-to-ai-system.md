---
icon: server-cog
---

# The LLM Server Architect

> *Working notes — May 2026. After fifteen years building distributed systems
> we know how to reason about, the LLM is the first new primitive in a
> decade that genuinely breaks the playbook. This is the umbrella note for
> a series; each later piece dives into one pillar.*

---

## 1. Why a new architect role?

- The thesis in one paragraph: distributed-systems intuition still applies,
  but four new primitives mutate the constraints — model-serving fleets,
  agent runtimes, agent-aware identity, and AI-shaped reliability budgets.
- Who this is for: distributed-systems engineers, SREs, platform leads who
  are now being asked to "own the AI stack."
- What this is *not*: a model-training note, a prompt-engineering note, or
  an LLMOps shopping list.

## 2. The classical baseline (one page, no more)

A short table to anchor the reader, mirroring the format used in §3 of
the IAM doc:

| Classical distributed system | LLM-native distributed system |
| --- | --- |
| Stateless service + DB + cache | Stateless service + **model fleet** + vector store + KV cache |
| Latency budgeted in ms | Latency budgeted in **seconds**, dominated by token generation |
| Failure modes: timeout, 5xx, partition | Failure modes: **wrong answer**, hallucination, prompt injection, runaway loop |
| Principal: user or service account | Principal: user, service, **or ephemeral agent acting on behalf of one** |
| Capacity: CPU, RAM, IOPS | Capacity: **GPU/accelerator-hours, KV-cache memory, context tokens** |
| Cost per request: ~free | Cost per request: **measurable, sometimes dollars** |
| Idempotency by request ID | Idempotency is **not guaranteed** — same input, different output |

Goal of the table: by the end, the reader feels every row has shifted.

## 3. The four new pillars

Each subsection is intentionally shallow — it states the problem, names the
new primitives, and ends with "this gets its own article."

### 3.1 Pillar 1 — Distributed LLM serving

- What an LLM server actually is (vLLM / TGI / TensorRT-LLM / SGLang as the
  new "nginx + app server" layer).
- What's different from a normal stateless service:
  - GPU/accelerator-bound, not CPU-bound.
  - Stateful in a sneaky way: KV cache, prefix cache, LoRA adapters loaded
    in memory.
  - Routing matters at the *prefix* level, not just the request level.
  - Batching is not a nicety; it's the primary throughput lever.
- New questions: how do you load-balance across heterogeneous GPUs? How do
  you handle a model swap without dropping in-flight tokens? What's the
  right unit of autoscaling — replica, GPU, or KV-cache slot?
- Pointer: *"Distributed LLM Serving"* — future article.

### 3.2 Pillar 2 — Distributed agents

- An agent is the new "service": LLM + tools + memory + control loop.
- Multi-agent systems are the new "microservices" — same coordination
  problems (orchestration vs choreography, supervision, back-pressure)
  with two new wrinkles:
  - The control flow is generated at runtime by a model, not coded.
  - Inter-agent protocols are emerging (MCP for agent→tool, A2A for
    agent→agent) and don't yet compose.
- New questions: where does the planner live? How do you bound recursion
  depth? How do you cancel a task that has already spawned three
  sub-agents? What replaces the call graph?
- Pointer: *"Agent Topologies and Coordination"* — future article.

### 3.3 Pillar 3 — Identity, authorization, and agent governance

- One-paragraph teaser of the IAM piece already written; link to it.
- Hooks worth foreshadowing here so readers know it's coming:
  - The principal stack (user → orchestrator → worker → tool).
  - Why session-time auth is too coarse and per-tool-call auth is the new
    granularity.
  - Why static API keys are now a "machine-speed kill chain."
- Pointer: [*IAM in Multi- and Autonomous-Agent Systems*](iam-in-multi-and-autonomous-agent-systems.md) — already published.

### 3.4 Pillar 4 — Availability, resiliency, durability, performance

This is the pillar a distributed-systems engineer feels most at home in,
*and* the one that mutates the most. Quick hits, each one a future article:

- **Availability.** What does "available" mean when the model is up but
  giving wrong answers? Need a *correctness SLO*, not just an uptime SLO.
- **Resiliency.** New failure classes: prompt injection, tool-call loops,
  model regression on a silent provider-side update, context-window
  exhaustion, KV-cache eviction storms, GPU OOM under bursty traffic.
- **Durability.** Conversations, agent memory, vector indices, episodic
  state — none of this maps cleanly onto "the database." What's the
  source of truth, and what's a derived cache?
- **Performance.** TTFT vs. tokens/sec vs. end-to-end task completion.
  Tail latency at the *token* level is a new thing.
- **Cost as a first-class SLI.** A retry storm in a classical system
  costs CPU; in an agent system it costs dollars and can run away while
  on-call sleeps.

## 4. The throughline

Borrow the move from §2.1 of the IAM doc: state the *one structural fact*
that ties the four pillars together, so the series has a spine.

Candidate framing (pick one and own it):

> Classical distributed systems were built around the assumption that
> *behavior is coded and identity is stable*. LLM-native systems invert
> both: behavior is *generated at runtime by a model* and identity is
> *ephemeral and stacked*. Every pillar in §3 is a different shadow of
> that inversion.

## 5. What changes for the engineer

- Skills you keep: capacity planning, queueing theory, idempotency,
  observability, blast-radius thinking, change management.
- Skills you have to add: token economics, eval design, prompt-injection
  threat modeling, GPU scheduling, agent governance, model-version
  pinning and rollback, semantic caching.
- A short "where to start if you're a senior engineer being told to own
  this on Monday" reading list / checklist.

## 6. Series roadmap

A bulleted list pointing to each future deep-dive, with a one-line hook:

- *Distributed LLM Serving* — KV cache, prefix routing, GPU autoscaling.
- *Agent Topologies and Coordination* — orchestrator/worker, peer, pipeline.
- [IAM in Multi- and Autonomous-Agent Systems](iam-in-multi-and-autonomous-agent-systems.md) — already up.
- *Reliability for Probabilistic Systems* — correctness SLOs, eval-as-tests.
- *Cost as an SLI* — token budgets, runaway-agent kill switches.
- *Observability for Agent Stacks* — traces across model, agent, and tool.
