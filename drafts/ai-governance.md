---
icon: brain-circuit
---

# Authorize the Work, Not the Worker: Authentication & Authorization in Agentic Systems

> *Thesis: Classical IAM governs **workers** — it authenticates a principal and authorizes a session, at a single hop. Agentic systems require governing **work** — a task that travels across many hops, changing hands, intent, and authority along the way. Until the governed unit shifts from the worker to the work, you can't build an agent system that is safe, least-privileged, and auditable.*

*Scope note: this is a deep-dive on the authentication/authorization question for people deciding how to build or buy an agent platform. It assumes the landscape as background and won't re-derive it — for the mechanics (what an agent is, the confused-deputy class, SPIFFE/OBO/A2A internals, the vendor map, the full reference architecture) it links to the companion working notes, [IAM in Autonomous Multi-Agent Systems](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md) (hereafter **AMAS**). What's here is the argument those notes don't make: that the unit you govern has to change.*

---

## 1. The shift: from worker to work

Classical identity was built to answer a question about *workers*: who is this principal, and what is their session allowed to do? You authenticate once, you get a token, the token names you and carries a scope, and for the life of that session the answer to "what can you do" is fixed. That model held for thirty years because the thing on the other end of the credential was stable — a person at a keyboard, or a service that does one job forever.

An agent breaks the model in one specific place: **it chooses its next action at runtime.** The call an agent will make is not knowable when its credentials are issued — the plan is generated, token by token, by a model reacting to what just happened. Authority granted up front ("you may call these tools") and intent formed later ("I've decided to email the CFO the spend report") come apart. Nearly everything else that is hard about agent security descends from that one gap (AMAS lays out the broken assumptions in [§3](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#3-what-is-different-about-iam-in-amas-vs-classical-systems)).

Now put more than one agent in the loop. The same task passes through an orchestrator, a worker, a tool, maybe a sub-agent — and **the principal acting at each hop changes.** By the time a row is written to a database, several identities and at least one human consent are stacked on top of each other. The question is no longer "what is this *worker* allowed to do." It's "what is *this task* allowed to become, on whose authority, as it moves?" The governed unit has to shift from the worker to the work.

That shift needs a noun. Call it the **work-item**: a first-class object that travels with the task and carries everything an authorization or audit decision downstream will need —

```text
work-item {
  origin:        initiating user + the consent they actually gave
  actor stack:   who is acting right now, and the chain that got here
  intent:        the original task, distinct from the current sub-goal
  scope:         the authority still in play — and it should only ever narrow
  correlation:   one stable id, from origin through every hop
}
```

AMAS already names half of this — it argues the system must track the *active actor stack* rather than "who's logged in." The work-item is that actor stack plus the two things the stack alone doesn't capture: where the task came from (origin + consent) and where its authority is allowed to go (a scope that shrinks). Hold this object in mind; the rest of the article is just "what does it take to authenticate it, authorize it, and audit it."

## 2. Why classical standards answer the wrong question

The instinct is to reach for what we already run — SAML, OpenID Connect, OAuth. None of them is wrong; each is built to answer a question that isn't the one an agent poses.

**SAML and OpenID Connect authenticate a human and mint a session.** SAML is browser-SSO-shaped: it asserts, to a relying party, that a *person* signed in. OIDC modernizes the same idea with an ID token describing a human subject. Both are authentication events. Neither has a first-class notion of a non-human actor acting *on behalf of* that human three services deep — the subject is the person, full stop. The moment a bot continues the work after the login, these protocols have already said everything they have to say.

**OAuth is the closest, because it's actually about delegated authorization** — but it delegates *one hop*. The resource owner authorizes a client; the scope is fixed at issuance; the audience is a single resource; and a bearer token is ambient authority — whoever holds it wields it, with no binding to a task or an intent. [OAuth 2.1 tightens the edges](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#52-authorization-protocols-and-extensions) (PKCE everywhere, implicit and ROPC gone) but doesn't change the shape: authority is bound to *a principal at a moment*, not to *a task over time*.

That is the whole gap in one line. The standards bind authority to a principal at a moment; agentic work needs authority bound to a task as it moves.

| What a decision downstream needs | SAML / OIDC | OAuth 2.1 | Agentic work needs |
| --- | --- | --- | --- |
| A task identity that persists across hops | — | — | first-class |
| Multi-hop delegation (user → agent → agent → tool) | — | one hop | N hops, verifiable |
| Re-evaluation as the plan changes | session-time | issuance-time | per call |
| Consent that composes (origin survives downstream) | — | scope is flat | narrows, travels |
| Binding to *this* task / resource, not ambient | session-wide | `aud` = one resource | task + resource bound |
| Revocation that propagates in seconds | logout / expiry | token expiry | continuous (see §5) |

The blanks are not implementation gaps to be configured away. They're the shape of protocols designed for workers, applied to work.

## 3. The delegation chain: where governing the work is won or lost

This is where the thesis earns its keep. A single tool call against a single API is the easy case — OAuth plus a decent policy engine mostly handles it. The hard case, and the one that defines agentic systems, is the chain: **user → orchestrator → worker → tool → sub-agent**, with the acting principal changing at every hop.

AMAS documents the three ways that chain fails, with mechanics and a real-world example; I won't repeat them — the point here is that all three are the *same* failure, and the work-item is what names it.

- **Confused deputy, at scale** ([AMAS §4.1](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#41-the-confused-deputy-problem-at-scale), including the February 2026 Cline compromise). An agent holds the *union* of its tool credentials, and nothing structurally binds the high-privilege tool it *can* reach to the low-privilege request it was *given*. In work-item terms: the actor is acting, but the work-item's origin and scope aren't consulted, so the deputy acts on its own authority instead of the task's.
- **Intent drift** ([AMAS §4.2](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#42-delegation-chain-depth-and-intent-drift)). "Help with the budget" becomes "fetch FY26 spend," then "join with the HR roster," then "email the CFO." Every step is a plausible reading of the last; no single hop looks wrong. In work-item terms: the *current sub-goal* has wandered away from *origin intent*, and nothing is comparing the two.
- **Consent that doesn't compose** ([AMAS §4.2](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#42-delegation-chain-depth-and-intent-drift)). The user consented to the goal, not to the downstream action. Authority flows forward; the consent boundary doesn't. In work-item terms: each hop sees only its immediate predecessor, because origin + consent aren't riding along.

Read together, these aren't three bugs. They're one structural fact wearing three masks: **the work-item isn't a first-class thing, so nothing can bound it.** AMAS makes the same point at the system level in [§2.1](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#21-why-no-common-identity-fabric-is-load-bearing) — identity, consent, intent, and audit don't travel intact across plane boundaries. The work-item is the unit on which they could travel.

And that exposes the genuinely missing primitive. Classical scopes bound *which tools you may call*. Nothing bounds *how far a task may travel* or *what it may become*. An agentic authority model needs two properties that no deployed standard provides:

1. **A depth bound** — a delegation can only go so many hops before it must stop or escalate to a human.
2. **Monotonic scope narrowing** — authority can only shrink as the work-item moves; a downstream hop can never hold more than the hop that handed it the task.

Neither is a config you're failing to set. They're capabilities the protocols don't have yet — which is why §4 and §5 are about assembling them from parts.

## 4. Authenticating the work

Authenticating the *work* means making the work-item's actor stack verifiable end to end — every hop should be able to prove who is acting and on whose authority. Today you can authenticate the links; you can't yet authenticate the chain. Two pieces do the link-level job:

- **The worker needs its own attested identity.** This is [SPIFFE / WIMSE](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#51-workload-identity-primitives): short-lived, cryptographically attested credentials with no static secrets. The reason this matters more for agents than for ordinary services is in [AMAS §4.3](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#43-static-credentials-dont-survive-contact-with-autonomy) — a leaked static key in a world of autonomous, API-speed callers is a machine-speed kill chain. Attested, expiring identity is the floor.
- **The user has to be threaded through.** [OAuth's On-Behalf-Of draft for AI agents](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#52-authorization-protocols-and-extensions) puts the user in `sub` and the agent in the `act` claim, so the access token and the audit log both carry a verifiable *user → agent* delegation. This is the single most important building block — and its limit is exactly the chain problem: OBO signs *one* link cleanly. Multi-hop (`act` nested inside `act`) is still draft territory ([Applier-OBO](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#52-authorization-protocols-and-extensions)).

Agent-to-agent handoffs ride on [A2A](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#2-how-do-multi-agent-systems-integrate-with-other-systems) — Agent Cards for discovery, signed task IDs per collaboration — and pair naturally with OBO to carry delegation across the hop. The honest status: a single hop of the work-item can be authenticated today with SPIFFE + OBO; the *chain* of hops cannot yet be verified end to end with anything standardized. That missing chain is the open problem, not a missing integration.

## 5. Authorizing the work

Authentication tells you the actor stack is real. It says nothing about whether the action should happen — and **authorization is not identity** ([AMAS §4.6](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#46-authorization--identity)). An OAuth scope answers "*can* this agent call this API?" It does not answer "*should* this action run, on this data, in this context, against this policy, right now?" That question can only be answered against the live work-item.

Three consequences for how authorization has to work:

- **Per-call, not per-session.** Because the plan changes mid-task, authority has to be re-evaluated at every call, with the work-item as input: current intent measured against origin intent, target resource, data sensitivity, approval thresholds. This is the bet behind the [runtime authorization platforms](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#53-continuous-runtime-authorization) (Aembit, SGNL, Strata, and peers — evaluate the category, don't anoint a winner): least privilege computed per call, not fixed at token issuance.
- **Scope narrows along the chain.** This is the authorization-side counterpart to §3's depth bound. The policy engine enforces monotonic narrowing — a sub-agent's effective authority is bounded by the work-item that reached it, never widened. The research direction here is [capability tokens versus ambient authority](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#6-what-to-watch-next): unforgeable, per-action grants the agent must present, instead of broad credentials it merely holds. Expect it first in finance and healthcare.
- **Trust has to be revocable in seconds.** An agent token can easily outlive the reason to trust it. [CAEP / Shared Signals](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#53-continuous-runtime-authorization) lets "this user is locked / this token is revoked / this device is compromised" propagate in seconds rather than at expiry — so authorization is continuous, not a one-time gate.

The pattern: identity says who's in the chain, runtime authorization decides each step, narrowing enforces the chain can't gain power, and continuous signals can pull the plug. All four read from the work-item.

## 6. Auditing the work

Audit is where the work-item pays for itself. The forensic question after an incident is brutal: *which human consented to what, which agent acted, which model version produced the plan, which tool was called, with which token, against which resource, at which time?* Today, as [AMAS §4.8](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#48-auditability-and-non-repudiation) describes, answering it is a heroic stitching exercise across the IdP, the A2A traces, the MCP server logs, and the model provider — because no plane carries the others' identifiers.

The work-item collapses that effort to a query. If a single **correlation id** is minted at origin and carried, intact, through every hop and into every log line, reconstructing the chain stops being archaeology and becomes a `WHERE correlation_id = …`. Non-repudiation is then a property you *designed in*, not one you reverse-engineer under pressure. The lesson for builders is blunt: if the work-item is first-class, audit is nearly free; if it isn't, audit is impossible at exactly the moment you need it.

## 7. A decision framework for architects

The standards won't be finished before you have to ship. So the practical question isn't "what's the perfect stack" — it's "what do I require now, what do I build around, and what do I defer." Map each to the work-item and the choices fall out.

**Require today** — table stakes; refuse a platform that misses these:

- Every agent has its own first-class identity (not a borrowed human account, not a shared service account).
- No static, long-lived secrets — attested, short-lived credentials only.
- Every tool call carries verifiable user context (OBO `act` claim or equivalent), so the work-item's origin survives the first hop.
- Authorization is evaluated per call, not per session.
- A single correlation id ties the whole chain together in the logs.

**Build around the gaps** — not standardized yet, so it's on you to enforce in your orchestration layer:

- Bound delegation **depth** — a hard limit past which the chain escalates to a human.
- Enforce **monotonic scope narrowing** — downstream never exceeds upstream.
- Make **consent composable** — carry origin intent forward and check current sub-goal against it.

**Defer / watch** — real, but not 2026 procurement criteria ([AMAS §6](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#6-what-to-watch-next) tracks these): DID/VC for cross-org agent trust, capability tokens going mainstream, and regulation (EU AI Act implementing acts, sectoral rules) that will eventually mandate demonstrable identity and audit chains.

**The five questions to put to any platform or vendor.** Each maps to a section above; vague answers are the finding:

1. Does **every** tool call carry the originating user's context — or does the agent call with its own credential? *(§4)*
2. Can you **bound delegation depth**, and what happens at the limit? *(§3)*
3. Is authorization decided **per call or per session**? *(§5)*
4. Is revocation **seconds or token-expiry**? *(§5)*
5. Can you reconstruct the **full chain — user, agents, model, tool, resource — from one query**? *(§6)*

For the end-state these add up to, use the consensus [reference architecture in AMAS §5.6](../AI_ML/misc/iam-in-multi-and-autonomous-agent-systems.md#56-reference-architecture-distilled) rather than redrawing it — human IdP → attested agent identity → delegation carried in `act` → resource-bound tool tokens → per-call policy engine → continuous evaluation. Read top to bottom, that diagram is just the work-item moving through the system with its identity, consent, and scope intact at every layer.

## Close

Agent identity is becoming a first-class, runtime-evaluated, auditable object — distinct from both human identity and the service-account hacks we reached for first. But the identity of the worker was never the hard part. The hard part is the **work**: a task that crosses hops, changes hands, and drifts from what was asked, while authority and consent have to follow it intact.

Govern the work-item — give it an origin, a verifiable actor stack, an intent it's checked against, a scope that only narrows, and an id that ties the whole chain together — and authentication, authorization, and audit stop being three unsolved problems and become three views of one object. Get that right and you can run agents at real autonomy without giving up least privilege, audit, or sleep. **Authorize the work, not just the worker.**
