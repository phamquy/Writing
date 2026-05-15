---
icon: brain-circuit
---

# IAM in Autonomous Multi-Agent Systems (AMAS)

> *Working notes — May 2026. Identity is now the load-bearing wall of agent security. Every other control (sandboxing, prompt-injection defense, tool-use policy) eventually depends on the question: "who is acting, on whose behalf, and with what authority right now?"*

---

## 1. What is an autonomous multi-agent system?

### 1.1 What is an AI agent?

An AI agent is a software system that wraps an LLM (or other reasoning model) in a control loop — *perceive → reason → plan → act → observe → adapt* — and gives it the ability to call **tools** (APIs, databases, code execution, other software) to make real-world changes on behalf of a user or an organization. Two properties separate an agent from a chatbot:

1. **It performs actions, not just produces text.** A chatbot describes the work; an agent does the work — opening tickets, moving money, writing to production systems.
2. **It decides what to do next.** Each turn of the loop is a runtime decision informed by what just happened, not a pre-coded workflow.

That second property is what breaks classical IAM: the agent's *next call* is not knowable when its credentials are issued.

### 1.2 What is a multi-agent system?

A multi-agent system (MAS) is several agents — often with specialized roles (planner, retriever, coder, reviewer, executor) — operating in the same environment and coordinating to solve a task that a single agent cannot, or cannot solve well. Coordination is usually one of:

- **Hierarchical / orchestrator-worker** — a supervisor agent decomposes a goal and farms sub-tasks out to workers.
- **Peer / market** — agents negotiate, bid, or vote.
- **Pipeline** — output of agent A is the input of agent B.

The IAM-relevant fact is that *the principal acting at each hop changes*. The user authorizes the orchestrator; the orchestrator delegates to a worker; the worker calls a tool. By the time a row gets written to a database, three identities and at least one human consent are stacked on top of each other.

### 1.3 What is an autonomous AI agent?

"Autonomous" is a spectrum, not a binary. Useful framing:

| Level | Description | Human in the loop? |
| --- | --- | --- |
| L1 — Assist | Suggests; human executes. | Every action |
| L2 — Co-pilot | Acts on narrow tools; human approves. | Per action |
| L3 — Delegated | Completes whole tasks; human approves at task start. | Per task |
| L4 — Autonomous | Operates open-endedly within a policy envelope; human is out of the loop until something escalates. | Exception only |
| L5 — Self-directed | Sets its own sub-goals, allocates its own resources, recruits other agents. | Rare / oversight only |

An **autonomous multi-agent system (AMAS)** lives at L3+ across multiple collaborating agents. That combination is what this document is about, because L3+ is where IAM stops being about *authenticating sessions* and starts being about *governing agency*.

---

## 2. How do multi-agent systems integrate with other systems?

There are three integration surfaces, and each one is its own identity boundary:

**(a) Agent ↔ Tools / Resources.** The dominant pattern in 2026 is the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/), which standardizes how an agent discovers and invokes tools, prompts, and resources on a remote server. As of the June 2025 spec revision, MCP servers are formally **OAuth 2.1 resource servers**, must publish protected-resource metadata, and clients must use Resource Indicators ([RFC 8707](https://datatracker.ietf.org/doc/html/rfc8707)) so a token issued for server X cannot be replayed at server Y.

**(b) Agent ↔ Agent.** [Google's Agent2Agent (A2A) protocol](https://github.com/a2aproject/A2A), donated to the Linux Foundation in 2025, is the closest thing to a winner. Agents publish an *Agent Card* — a machine-readable manifest of identity, skills, supported modalities, and security requirements — and authenticate to one another using the same schemes as OpenAPI: API keys, OAuth 2.0, OpenID Connect. Short-lived tokens and signed task IDs scope each collaboration without hardcoded secrets.

**(c) Agent ↔ Humans / Enterprise systems.** The agent has to plug into the same SSO, RBAC, audit, and DLP systems the rest of the company uses — Okta, Entra ID, Ping, AWS IAM, Google Cloud IAM, ServiceNow, Salesforce. This is where most of the *real* pain lives, because none of those systems were designed with a non-human caller that picks its own next action.

A useful mental model is that MCP solves *agent → tool*, A2A solves *agent → agent*, and existing enterprise IAM still has to solve *human → agent*. Today these three planes do not share a common identity fabric, which is the ultimate source of most of the problems in §4.

### 2.1 Why "no common identity fabric" is load-bearing

Saying these planes "don't share a fabric" is not just a tidiness complaint — it shows up as a specific, repeating pattern: at every plane boundary, identity, intent, consent, or audit gets dropped, coerced, or widened. The same root cause re-surfaces under different names in OWASP, in incident reports, and in vendor pitches. Concretely, the gaps are:

- **Different identity primitives that do not encode each other.** Enterprise IAM speaks OIDC subjects, employee IDs, group memberships, and device posture. A2A speaks Agent Cards and OpenAPI-style auth schemes. MCP speaks OAuth 2.1 access tokens scoped to a single resource server. None of these natively carries the others. So when an action crosses planes, somebody hand-rolls a mapping — and the mapping is where service accounts, shared bearer tokens, and "the agent's own credential" sneak in to fill the gap.
- **Trust roots that don't compose.** The IdP that Okta or Entra signs against, the SPIFFE trust domain that issues the agent's SVID, and the authorization server an MCP resource trusts are three independent roots. Verifying a chain end-to-end requires bridging between them, and bridges historically get implemented as long-lived API keys "because it was easier."
- **The user drops out at the first hop.** When a human signs into an agent app via the IdP, the agent then calls an MCP tool using its *own* credential. Absent OBO (still an IETF draft), the tool only sees the agent — never the user. That means every MCP server is forced into a confused-deputy posture *by default*: it can authorize only what the agent itself is allowed to do, which is the union of every user it has ever served. ASI03 (Identity & Privilege Abuse) and the entire confused-deputy class in §4.1 are downstream of this single gap.
- **Consent gets coarser with every translation.** The user's OIDC consent ("read my email for the next hour") becomes an OAuth scope at the MCP boundary ("`gmail.read`"), then becomes ambient authority inside the agent's loop ("the agent can call `gmail.read` whenever it decides to"). By hop 3 — agent → sub-agent → tool — the original "for the next hour, for this task" has eroded into "anything within scope." Intent drift (§4.2) is the visible symptom; the cause is that consent semantics don't ride along through the protocol switch.
- **Audit is fragmented across three logs with no shared correlation ID.** The login event lives in the IdP. The agent-to-agent handoff lives in A2A traces (or doesn't, if the agents are on different platforms). The tool call lives in the MCP server's log. The model invocation lives at the model provider. Reconstructing "which user, via which agent, with which plan, called this tool" is the heroic stitching exercise of §4.8 — and it's heroic precisely because no plane carries the others' identifiers as first-class fields.
- **Lifecycle events don't propagate.** Off-boarding a user in the IdP doesn't automatically revoke the agent tokens that were minted on their behalf. Rotating an agent's SPIFFE identity doesn't invalidate the Agent Cards that other agents pinned. Revoking an MCP integration doesn't tell the orchestrator to stop trying. Each plane runs on its own clock; CAEP/SSF (§5.3) is the proposed cross-cutting fix, but it isn't yet wired into most agent runtimes.
- **Discovery and inventory are split three ways.** Agents register with cloud IAM (Entra Agent ID, Google Cloud Agent Identity), with A2A registries (Agent Cards), and with MCP servers (as OAuth clients). Security and IT have no single pane of glass to answer "every agent that can touch our data, and on whose authority." That is why §4.5 (discovery / lifecycle) is its own challenge rather than a footnote.
- **Authorization decisions are made with partial visibility.** The MCP resource server only sees its OAuth token. It doesn't see what user is upstream, what other agents are in the chain, what the original task was, or what other tools have already been called. Runtime authorization engines (Aembit, SGNL, Strata, etc., in §5.3) exist precisely to re-stitch this context — but they're working around the fact that the underlying protocols don't carry it.

The throughline: most of the problems in §4 are not independent bugs to be patched separately. They are different shadows of the same structural fact — that *identity, consent, intent, and audit do not travel intact across plane boundaries*. Most of the emerging standards and products in §5 are, ultimately, attempts to build that missing fabric: OBO threads the user through, SPIFFE/WIMSE unifies the agent identifier, CAEP/SSF unifies revocation signals, and runtime authorization engines unify the policy decision. None of them solve it in isolation; the unsolved part is making them compose.

---

## 3. What is different about IAM in AMAS vs. classical systems?

Classical IAM rests on assumptions that quietly break in agentic settings:

| Assumption (classical IAM) | What an AMAS does instead |
| --- | --- |
| A principal is either a human or a long-lived service. | A principal is an *ephemeral agent instance* spun up for a task and gone in minutes. |
| Authorization is decided once, at session start. | Authorization should be re-evaluated *per tool call* because the agent's plan changes mid-session. |
| The caller's intent matches the credential's scope. | The agent's intent is generated by an LLM at runtime; intent and scope can diverge with each token. |
| Delegation is one hop (user → app). | Delegation is N hops (user → orchestrator → worker → tool → sub-agent...). |
| Identities are provisioned by IT before they are used. | Agents can spawn other agents on the fly — "just-in-time" identity is the norm. |
| Audit logs answer "who did this?" with a username. | Audit must answer "which user, via which agent chain, under whose consent, with what plan, called this tool?" |
| Static credentials (API keys) are acceptable for service accounts. | Static credentials become a *machine-speed kill chain* if leaked, because agents move at API speed and don't sleep. |

Two consequences worth calling out explicitly:

1. **Non-human identities (NHIs) now dwarf human identities.** Reports in 2026 put the ratio at 40:1 to 100:1, and 500:1 in highly automated shops, with the average enterprise carrying ~250k NHIs across cloud — ~71% un-rotated and ~97% over-privileged. AMAS is the accelerant.
2. **The "principal" is no longer a stable noun.** The same agent *instance* may legitimately act on behalf of different users in different turns of its life, and may compose authority from multiple delegation chains. IAM has to track the *active actor stack*, not just "who's logged in."

---

## 4. What challenges does AMAS introduce?

### 4.1 The confused-deputy problem, at scale

The classic confused-deputy bug — a privileged intermediary tricked into acting outside the caller's actual authority — is the *default architecture* of an agent. The agent has the union of its tool credentials; the user's intent reaches it as natural language; nothing structurally constrains the agent from using a high-privilege tool to satisfy a low-privilege request. The February 2026 [Cline coding-assistant compromise](https://dev.to/claude-go/the-confused-deputy-problem-just-hit-ai-agents-and-nobodys-scanning-for-it-384f), where a malicious GitHub issue title coerced an authenticated Claude session into installing an attacker package on ~4,000 developer machines, is the canonical real-world example.

### 4.2 Delegation chain depth and intent drift

Each hop (A→B→C→tool) is a chance for intent to shift. "Help me with the budget" gets refined into "fetch FY26 spend," then "join with HR roster," then "email the join to the CFO." Every step is plausible; none was explicitly consented to. There is, today, no widely deployed way to bound *how far* a delegation may travel or *what it may become* along the way.

Three things make this especially dangerous:

- **No single hop looks wrong.** Each refinement is a reasonable interpretation of the previous one, so neither human review nor a per-call policy check has a clean signal to fire on. The harm is *cumulative semantic drift*, not any one violation — and most authorization systems only evaluate one call at a time.
- **Consent doesn't compose.** The user consented to "help with the budget," not to "email PII to the CFO." But every downstream agent sees only the *immediately upstream* request, which looks legitimate in isolation. Authority flows forward; the original consent boundary doesn't — so by hop 3 the agent is acting with the user's credentials on a task the user never approved.
- **It's the confused-deputy problem, compounded.** §4.1 already establishes that an agent holds the union of its tool credentials. Add multi-hop delegation and you get *transitive* confused deputy: agent C uses A's authority to do something A would have refused, because each intermediary only enforced its local view of intent. There is no equivalent of OAuth's `aud`/`scope` that travels with the *task* and narrows on each hop.

The deeper structural point: classical IAM scopes bound *what tools you can call*; nothing bounds *how far a task may evolve from its original intent*. That is a missing primitive, not a missing config.

### 4.3 Static credentials don't survive contact with autonomy

API keys, long-lived service account tokens, and `.env` secrets were acceptable in a world where a leak meant a human attacker had hours to exploit it. An over-privileged agent token leaked into a model's context, or into a log, becomes a machine-speed kill chain — no malware, no C2, just an LLM with a credential and a broad scope.

### 4.4 The agent-identity gap

[Strata's 2026 research](https://www.strata.io/blog/agentic-identity/the-ai-agent-identity-crisis-new-research-reveals-a-governance-gap/) found only ~18% of security leaders are highly confident their IAM can manage agent identities; only ~22% treat agents as first-class identity-bearing entities. Most orgs reuse a human's account or stand up a generic service account, both of which destroy the audit trail and break least privilege. Top reported concerns:

- Sensitive data exposure (55%)
- Unauthorized actions (52%)
- Credential misuse (45%)
- Lack of identity standards (45%)
- Inability to discover or register agents (40%)

### 4.5 Discovery, inventory, and lifecycle

You can't govern what you can't enumerate. Agents are spawned by developers, by other agents, by SaaS products (Copilot, ServiceNow, Salesforce, Bedrock AgentCore, Vertex Agent Builder...) — often without ever touching IAM. There is no equivalent of an HR system saying "this agent has been off-boarded."

### 4.6 Authorization ≠ identity

Even with perfect identity, OAuth scopes answer *"can the agent call this API?"* — they do not answer *"should the agent execute this action, on this data, in this context, against this policy, right now?"* Business policy, data residency, approval thresholds, and intent-vs-scope checks live above OAuth and need a separate runtime authorization layer.

### 4.7 Memory and context as a new attack surface

An agent's long-term memory and context window can carry tokens, PII, and prior consent decisions across sessions. Prompt injection that exfiltrates memory is functionally credential theft. Few IAM systems treat "context contents" as a governed resource.

### 4.8 Auditability and non-repudiation

Forensics needs to reconstruct: *which human consented to what, which agent acted, which model version produced the plan, which tool got called, with which token, against which resource, at which time*. Stitching that together across MCP servers, A2A hops, model providers, and enterprise SaaS is — today — a heroic effort.

---

## 5. Current state and emerging solutions

The landscape is moving fast and the pieces don't yet compose into a single playbook. The honest summary: **identity is the agentic AI problem nobody has fully solved yet**. What exists:

### 5.1 Workload identity primitives

- **[SPIFFE / SPIRE](https://spiffe.io/)** — cryptographically verifiable workload identity (SVIDs as X.509 or JWT) without long-lived secrets. Already operational at hyperscale (Uber attests billions/day). HashiCorp, CyberArk, and Solo.io have all published patterns for using SPIFFE IDs as the canonical identifier for agents.
- **[WIMSE](https://datatracker.ietf.org/wg/wimse/about/)** (Workload Identity in Multi-Service Environments) — IETF working group generalizing SPIFFE-style identifiers across trust domains. Likely to underlie cross-cloud and cross-org agent identity.

### 5.2 Authorization protocols and extensions

- **OAuth 2.1** — the 2025 consolidation of OAuth 2.0 + the security BCP. Mandates PKCE for public clients; removes implicit and ROPC. This is the floor.
- **MCP Authorization spec** — OAuth 2.1 + PKCE with Resource Indicators ([RFC 8707](https://datatracker.ietf.org/doc/html/rfc8707)) baked in; servers publish protected-resource metadata; in the [2026 MCP roadmap](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/), active proposals include SEP-1932 (DPoP) and SEP-1933 (Workload Identity Federation).
- **[OAuth 2.0 OBO for AI Agents](https://www.ietf.org/archive/id/draft-oauth-ai-agents-on-behalf-of-user-01.html)** — IETF draft that adds `requested_actor` and `actor_token` parameters so the consent screen, the access token, and the audit log all carry a *user → agent* delegation that downstream resource servers can verify. The token's `sub` is the user; the `act` claim is the agent.
- **Agent collaboration extensions** (e.g. [Applier-On-Behalf-Of](https://www.ietf.org/archive/id/draft-song-oauth-ai-agent-collaborate-authz-00.html)) — early drafts for multi-hop agent-to-agent delegation chains.
- **A2A** — uses OpenAPI-style auth schemes; pairs naturally with OBO for cross-agent delegation.

### 5.3 Continuous, runtime authorization

- **[CAEP / Shared Signals Framework](https://openid.net/specs/openid-caep-1_0-final.html)** — finalized by the OpenID Foundation in late 2025. Lets identity providers, resource servers, and agent platforms publish/subscribe Security Event Tokens so that "this token is now revoked / this user is locked / this device is compromised" propagates in seconds rather than at token expiry. Critical for agents because their tokens may live longer than the user's context for trusting them.
- **Runtime authorization platforms** — Aembit, Astrix, Strata, Curity, Apono, Permiso, SGNL, WitnessAI, etc. The shared bet: least privilege must be evaluated *per call*, with the agent's current intent, target resource, data sensitivity, and approval thresholds as inputs — not fixed at token issuance.

### 5.4 Cloud and platform offerings

- **Google Cloud — Agent Identity** (GA for Agent Runtime, preview for Gemini Enterprise Agent Platform), with IAM Allow/Deny and Principal Access Boundary support specifically for agents.
- **Microsoft — Entra Agent ID** + the [Agent Governance Toolkit](https://github.com/microsoft/agent-governance-toolkit), policy enforcement, zero-trust, sandboxing for Copilot agents.
- **AWS — Bedrock AgentCore Identity**, with on-behalf-of patterns over IAM Identity Center.
- **Auth0 + Google Cloud A2A partnership**, Ping Identity's Identity-for-AI, Okta's agent-security work — all converging on a similar pattern: agent gets its own first-class identity, every call carries delegated user context, runtime engine checks policy.

### 5.5 Standards bodies and frameworks

- **NIST / NCCoE** — published the concept paper ["Accelerating the Adoption of Software and AI Agent Identity and Authorization"](https://csrc.nist.gov/pubs/other/2026/02/05/accelerating-the-adoption-of-software-and-ai-agent/ipd) (Feb 5, 2026; comment period closed Apr 2, 2026), and announced the [AI Agent Standards Initiative](https://www.nist.gov/caisi/ai-agent-standards-initiative) out of CAISI. Focus areas: identification, authorization, audit, non-repudiation, and prompt-injection mitigation.
- **OWASP — [Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)**. Three of the top four (ASI02 Tool Misuse, ASI03 Identity & Privilege Abuse, ASI04 Delegated Trust) are identity-shaped.
- **Gartner** — named "IAM Adapts to AI Agents" a top-6 cybersecurity trend for 2026.
- **W3C DIDs / Verifiable Credentials** — research direction ([arXiv 2511.02841](https://arxiv.org/abs/2511.02841)) for cross-org agent identity where no single IdP is trusted by both sides, useful for agent-to-agent trust on the open web.
- **Grantex** — an OAuth-2.0-inspired protocol purpose-built for AI agents with cryptographic agent identity (DIDs), scoped + time-limited grant tokens, immutable audit trails, and explicit multi-agent delegation chains. Niche today, but a useful design reference.

### 5.6 Reference architecture, distilled

Pulling the above together, the consensus reference architecture in 2026 looks roughly like:

```text
┌────────────────────────────────────────────────────────────────────┐
│  Human user (IdP: Okta / Entra / Auth0 / Ping ...)                 │
└───────────────────┬────────────────────────────────────────────────┘
                    │   OAuth 2.1 + OBO (sub=user, act=agent)
                    ▼
┌────────────────────────────────────────────────────────────────────┐
│  Agent identity (SPIFFE / WIMSE / cloud-native agent ID)           │
│  ─ short-lived attested credentials, no static secrets             │
└───────────────────┬────────────────────────────────────────────────┘
                    │   A2A (Agent Card, signed task IDs)
                    ▼
┌────────────────────────────────────────────────────────────────────┐
│  Other agents — delegation chain carried in act / actor_token      │
└───────────────────┬────────────────────────────────────────────────┘
                    │   MCP (OAuth 2.1 + PKCE + RFC 8707)
                    ▼
┌────────────────────────────────────────────────────────────────────┐
│  Tools / resources — token bound to *this* MCP server              │
└───────────────────┬────────────────────────────────────────────────┘
                    │
                    ▼
┌────────────────────────────────────────────────────────────────────┐
│  Runtime authorization engine (Aembit / SGNL / Strata / cloud-     │
│  native PEP) ─ checks policy per call: who, on whose behalf,       │
│  what, on what data, in what context                               │
└───────────────────┬────────────────────────────────────────────────┘
                    │   Security Event Tokens (CAEP / SSF)
                    ▼
┌────────────────────────────────────────────────────────────────────┐
│  Continuous evaluation — revoke / step-up / quarantine in seconds  │
└────────────────────────────────────────────────────────────────────┘
```

---

## 6. What to watch next

Things that are shaping up to matter over the next 12–24 months:

1. **OBO for AI Agents → RFC.** The IETF draft is the most likely short-term standard to produce a *vendor-neutral* user-→-agent delegation primitive. Watch for the `act` claim becoming load-bearing in audit pipelines.
2. **Multi-hop delegation standards.** The hard problem after OBO is bounded *agent-→-agent-→-agent* chains: depth limits, scope-narrowing along the chain, and verifiable consent at every hop. Drafts like Applier-OBO and Grantex are early signals.
3. **MCP DPoP and Workload Identity Federation (SEP-1932 / SEP-1933).** Moves MCP from "bearer tokens with OAuth" toward proof-of-possession and federated workload identity. Reduces blast radius of token leaks.
4. **NIST AI Agent Standards Initiative + NCCoE practice guide.** Likely to produce the first widely-cited *enterprise* reference architecture, which procurement teams can hold vendors to.
5. **CAEP/SSF adoption by agent platforms.** Critical for closing the "long-lived agent token" problem. Watch which agent runtimes (Bedrock AgentCore, Vertex, Copilot Studio, etc.) become SSF transmitters and receivers.
6. **First-class "Agent Identity" SKUs from IdPs.** Entra Agent ID, Google Cloud Agent Identity, Okta + Auth0 agent products. The market is converging on agent-as-principal as a billable, governed object — not a service-account hack.
7. **Memory and context governance.** Expect new controls — and probably a new acronym — for "what the agent is allowed to remember about whom, for how long, and who can read it." Today this is the under-defended flank.
8. **Capability-based authorization vs. ambient authority.** Research (DeepMind Feb 2026; capability-security revival in [the Vectors blog](https://niyikiza.com/posts/capability-delegation/)) is pushing toward agents holding *unforgeable capability tokens* per action, instead of ambient credentials. Most likely to appear first in high-stakes domains (finance, healthcare).
9. **DID/VC for cross-organization agent trust.** Useful where two companies' agents must transact without a shared IdP. Not a 2026 mainstream story, but a 2027–2028 one.
10. **Regulation catching up.** EU AI Act implementing acts, US executive orders, and sectoral regulators (financial, healthcare) starting to require demonstrable identity, audit, and human-accountability chains for autonomous agents. The first enforcement actions will shape the market faster than any standard.

The throughline across all of these: **agent identity is becoming a first-class, governed, auditable, runtime-evaluated thing, distinct from both human identity and traditional service accounts.** The orgs that get this right early will be able to deploy autonomous agents at L3+ without giving up on least privilege, audit, or sleep.

---

## Sources

- [What Is Agentic AI? Complete Guide To Autonomous AI Systems In 2026 — AITUDE](https://www.aitude.com/what-is-agentic-ai-complete-guide-to-autonomous-ai-systems-in-2026/)
- [Multi-Agent Systems & AI Orchestration Guide 2026 — Codebridge](https://www.codebridge.tech/articles/mastering-multi-agent-orchestration-coordination-is-the-new-scale-frontier)
- [Agentic AI, explained — MIT Sloan](https://mitsloan.mit.edu/ideas-made-to-matter/agentic-ai-explained)
- [The AI Agent Identity Crisis: A 2026 Guide — Strata](https://www.strata.io/blog/agentic-identity/the-ai-agent-identity-crisis-new-research-reveals-a-governance-gap/)
- [Identity Is the Agentic AI Problem Nobody Has Solved Yet — Resilient Cyber](https://www.resilientcyber.io/p/identity-is-the-agentic-ai-problem)
- [IAM for Agentic AI: The New Perimeter of Trust in 2026 — Aembit](https://aembit.io/blog/iam-agentic-ai/)
- [AI Agents Are Creating an Identity Security Crisis in 2026 — IANS](https://www.iansresearch.com/resources/all-blogs/post/security-blog/2026/02/24/ai-agents-are-creating-an-identity-security-crisis-in-2026)
- [Gartner IAM Summit 2026 recap — GitGuardian](https://blog.gitguardian.com/gartner-iam-summit-2026-identity-expanded-faster-than-most-programs-did/)
- [State of AI Agent Security 2026 Report — Gravitee](https://www.gravitee.io/blog/state-of-ai-agent-security-2026-report-when-adoption-outpaces-control)
- [Non-Human Identities (NHI): The Hidden Security Crisis — Protego](https://protego.me/blog/non-human-identities-nhi-ai-agent-security-2026)
- [SPIFFE — Secure Production Identity Framework for Everyone](https://spiffe.io/)
- [SPIFFE: Securing the identity of agentic AI and non-human actors — HashiCorp](https://www.hashicorp.com/en/blog/spiffe-securing-the-identity-of-agentic-ai-and-non-human-actors)
- [Agent Identity and Access Management — Can SPIFFE Work? — Solo.io](https://www.solo.io/blog/agent-identity-and-access-management---can-spiffe-work)
- [MCP Specs Update: All About Auth — Auth0](https://auth0.com/blog/mcp-specs-update-all-about-auth/)
- [MCP Authentication and Authorization — Stack Overflow Blog](https://stackoverflow.blog/2026/01/21/is-that-allowed-authentication-and-authorization-in-model-context-protocol/)
- [The 2026 MCP Roadmap — Model Context Protocol Blog](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/)
- [MCP, OAuth 2.1, PKCE, and the Future of AI Authorization — Aembit](https://aembit.io/blog/mcp-oauth-2-1-pkce-and-the-future-of-ai-authorization/)
- [Announcing the Agent2Agent Protocol (A2A) — Google Developers Blog](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)
- [Secure A2A Authentication with Auth0 and Google Cloud — Auth0](https://auth0.com/blog/auth0-google-a2a/)
- [A2A Protocol Specification](https://a2a-protocol.org/latest/specification/)
- [What is Agent2Agent Protocol (A2A)? — Ping Identity](https://developer.pingidentity.com/identity-for-ai/agents/idai-what-is-a2a.html)
- [OAuth 2.0 Extension: On-Behalf-Of User Authorization for AI Agents — IETF draft](https://www.ietf.org/archive/id/draft-oauth-ai-agents-on-behalf-of-user-01.html)
- [draft-oauth-ai-agents-on-behalf-of-user — IETF Datatracker](https://datatracker.ietf.org/doc/draft-oauth-ai-agents-on-behalf-of-user/)
- [OAuth2.0 Extension for Multi-AI Agent Collaboration: Applier-On-Behalf-Of — IETF draft](https://www.ietf.org/archive/id/draft-song-oauth-ai-agent-collaborate-authz-00.html)
- [OAuth's On-Behalf-Of flow for AI agents — WorkOS](https://workos.com/blog/oauth-on-behalf-of-ai-agents)
- [Grantex: OAuth-Inspired Authorization Protocol for AI Agents — Agent Wars](https://agent-wars.com/news/2026-03-15-grantex-delegated-authorization-protocol-ai-agents)
- [The Confused Deputy Problem Just Hit AI Agents — DEV Community](https://dev.to/claude-go/the-confused-deputy-problem-just-hit-ai-agents-and-nobodys-scanning-for-it-384f)
- [Confused Deputy Attacks on Autonomous AI Agents — Cloud Security Alliance](https://labs.cloudsecurityalliance.org/research/csa-research-note-ai-agent-confused-deputy-prompt-injection/)
- [Control the chain, secure the system: Fixing AI agent delegation — Okta](https://www.okta.com/blog/ai/agent-security-delegation-chain/)
- [Before you build agentic AI, understand the confused deputy problem — HashiCorp](https://www.hashicorp.com/en/blog/before-you-build-agentic-ai-understand-the-confused-deputy-problem)
- [DeepMind Study Proposes Rules for How AI Agents Should Delegate — The AI Insider](https://theaiinsider.tech/2026/02/17/deepmind-study-proposes-rules-for-how-ai-agents-should-delegate/)
- [Capabilities Are the Only Way to Secure Agent Delegation — Vectors](https://niyikiza.com/posts/capability-delegation/)
- [NIST/NCCoE concept paper: Accelerating the Adoption of Software and AI Agent Identity and Authorization](https://csrc.nist.gov/pubs/other/2026/02/05/accelerating-the-adoption-of-software-and-ai-agent/ipd)
- [Software and AI Agent Identity and Authorization — NCCoE](https://www.nccoe.nist.gov/projects/software-and-ai-agent-identity-and-authorization)
- [Announcing the AI Agent Standards Initiative — NIST](https://www.nist.gov/news-events/news/2026/02/announcing-ai-agent-standards-initiative-interoperable-and-secure)
- [Everything you should know about NIST's AI Agent Standards Initiative — WorkOS](https://workos.com/blog/nist-ai-agent-standards-initiative-explained)
- [OWASP Top 10 for Agentic Applications 2026 — OWASP GenAI](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)
- [The OWASP Agentic Top 10 2026 — Entro Security](https://entro.security/blog/the-owasp-agentic-top-10-2026-what-it-means-for-ai-agents-and-non-human-identities/)
- [OpenID CAEP 1.0 Final Specification](https://openid.net/specs/openid-caep-1_0-final.html)
- [SGNL — Shared Signals and CAEP final specs published](https://sgnl.ai/2025/09/sgnl-welcomes-the-publication-of-the-final-shared-signals-and-caep-specifications/index.html)
- [What's new in IAM: Security, governance, and runtime defense — Google Cloud](https://cloud.google.com/blog/products/identity-security/whats-new-in-iam-security-governance-and-runtime-defense/)
- [Authorization and Governance for AI Agents: Runtime Authorization Beyond Identity at Scale — Microsoft](https://techcommunity.microsoft.com/blog/microsoft-security-blog/authorization-and-governance-for-ai-agents-runtime-authorization-beyond-identity/4509161)
- [Microsoft Agent Governance Toolkit — GitHub](https://github.com/microsoft/agent-governance-toolkit)
- [Secure AI agents with Amazon Bedrock AgentCore Identity on Amazon ECS — AWS](https://aws.amazon.com/blogs/machine-learning/secure-ai-agents-with-amazon-bedrock-agentcore-identity-on-amazon-ecs/)
- [Enable Agentic AI Securely and Confidently — Ping Identity](https://www.pingidentity.com/en/solution/agentic-ai-identity.html)
- [Curity: runtime authorization for AI agents — CSO Online](https://www.csoonline.com/article/4158847/curity-looks-to-reinvent-iam-with-runtime-authorization-for-ai-agents.html)
- [AI Agents with Decentralized Identifiers and Verifiable Credentials — arXiv 2511.02841](https://arxiv.org/abs/2511.02841)
- [A Novel Zero-Trust Identity Framework for Agentic AI — arXiv 2505.19301](https://arxiv.org/html/2505.19301v1)
