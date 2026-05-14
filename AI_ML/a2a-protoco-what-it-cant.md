# A2A Protocol: What It Is, What It Isn't, and How to Fill the Gaps

The Agent2Agent (A2A) protocol is an open standard for communication between AI agents, originally contributed by Google and now governed by the Linux Foundation. It's frequently discussed alongside MCP as a foundational piece of the emerging agent ecosystem — but unlike MCP, which connects agents to *tools*, A2A connects agents to *other agents*.

After working through the spec in depth, a clear pattern emerges that's worth stating up front: **A2A is a deliberately minimal protocol that specifies the bilateral interaction envelope between two agents precisely, and leaves nearly everything else — governance, identity propagation, topology, memory, discovery, mid-flight steering — to higher layers.** Whether that's the right amount of protocol is genuinely debatable. Some argue A2A is too thin to be useful and doesn't standardize the things that actually matter. Others argue it's exactly thin enough to get broad adoption without dictating implementation choices.

This article walks through what A2A actually defines, where the gaps are, and the practical options for filling them.

---

## What A2A Is at the Wire Level

At its core, A2A defines how one agent (the *client*) can send a task to another agent (the *remote*), receive progress updates, and obtain a result — even when the remote is a black box built on a different framework by a different organization.

The protocol specifies:

- **Wire format**: JSON-RPC 2.0 over HTTP(S), with optional SSE for streaming and webhooks for push notifications.
- **A standardized method vocabulary**: `message/send`, `message/stream`, `tasks/get`, `tasks/cancel`, `tasks/pushNotificationConfig/set`, and a few others. Every compliant agent implements the same set.
- **A standardized object model**: `Task`, `Message`, `Part`, `Artifact`, `AgentCard`, plus `contextId` and `taskId` for correlation.
- **A task lifecycle** with defined states: `submitted`, `working`, `input-required`, `auth-required`, `completed`, `failed`, `canceled`, `rejected`, `unknown`.
- **An AgentCard discovery primitive**: a JSON document at `/.well-known/agent-card.json` describing an agent's identity, endpoint, skills, supported modes, and auth requirements.
- **Auth hooks**: agents declare their authentication scheme in the AgentCard, and clients satisfy it using standard HTTP auth (OAuth 2.0, OIDC, API keys).

### Is this really different from HTTP?

It's worth addressing this honestly because it's the question that exposes whether A2A is a "real" protocol or just HTTP with conventions.

Mechanically, A2A *is* HTTP. If you sniff the network, an A2A request is indistinguishable from any other JSON-RPC-over-HTTP API call. There's no magic on the wire. You could implement an A2A client with `curl` or `fetch()`.

So mechanically, A2A communication *is* HTTP communication. The question is: what does A2A add on top, and does it justify being called a "protocol" rather than "a convention"?

A few things HTTP alone doesn't give you:

**A standardized method vocabulary.** HTTP gives you GET/POST/PUT/DELETE and lets you design any URL structure and request body you want. Every REST API invents its own shape: `POST /api/v1/orders`, `PUT /customers/{id}`, etc. There's no agreement across APIs about what method names mean. A2A specifies a fixed set of JSON-RPC methods that every compliant agent must implement. Instead of every agent having a bespoke API, every agent has *the same* API. A client written for agent A works against agent B with no code changes — that's the whole point.

**A standardized object model.** HTTP doesn't define what a "task" is, what a "message" looks like, or what an "artifact" is. Every REST API invents these. A2A defines them with specific schemas. Two A2A agents agree on what these things are.

**A standardized lifecycle.** HTTP is fundamentally request/response with no notion of long-running work. If you want a long-running job over HTTP, every API invents its own pattern (polling, webhooks, SSE, long-polling). A2A specifies *one* pattern: tasks have states, you can stream them or poll them or get pushed to, all using the same method names and state machine.

**A standardized discovery mechanism.** HTTP gives you nothing for discovery. You read docs and hardcode URLs. A2A says: every agent exposes an AgentCard at `/.well-known/agent-card.json` with a defined schema. A client can fetch any A2A agent's card and know what it does, what auth it needs, what input modes it accepts.

**Semantic conventions for the interaction.** HTTP is content-agnostic. POST means "submit something" but doesn't specify what or how the server should treat it. A2A says: this is *agent-to-agent* communication. The payload semantics are tasks, the response semantics are artifacts, the conversation semantics are contexts. It's an *opinionated* layer that says "we're doing agent collaboration, not arbitrary API calls."

### The right analogy

A useful comparison: **A2A is to HTTP what REST is to HTTP, or what gRPC is to HTTP/2.** REST isn't a different protocol from HTTP — it's HTTP plus a set of architectural constraints (resources, statelessness, uniform interface). You can do REST with `curl`. But REST gives you a shared vocabulary so that everyone's APIs *feel* similar. A2A is HTTP plus a more specific set of constraints aimed at agent communication. You can do A2A with `curl`. But A2A gives you a shared vocabulary so that any agent can talk to any other agent without bespoke integration.

Or another angle: gRPC is "just" HTTP/2 with protobuf and a defined RPC pattern. But nobody would say "what's the difference between gRPC and HTTP/2?" — gRPC's value is the *standardization*, not the transport.

Concretely, here's what changes when you move from "agent A calls agent B's bespoke REST API" to "agent A calls agent B over A2A":

| Without A2A | With A2A |
|---|---|
| A reads B's custom API docs | A fetches B's AgentCard |
| A writes B-specific client code | A uses a generic A2A client library |
| B invents endpoints (`POST /summarize`) | B implements `message/send` with a skill called "summarize" |
| B invents response shapes | B returns standard Task/Message/Artifact objects |
| B invents how to stream progress | B uses A2A's SSE streaming on `message/stream` |
| B invents how to cancel | B implements `tasks/cancel` |
| B invents conversation tracking | B uses `contextId` |
| Adding agent C means writing a new client | Adding agent C means... nothing, same client works |

The value isn't in any single feature — it's in **not having to negotiate any of these things bilaterally for every new agent pair**.

### When the distinction matters, and when it doesn't

There are moments where you'd genuinely feel "this isn't just HTTP":

1. **Onboarding a new agent.** With a custom HTTP API, integrating a new vendor's agent means days of work. With A2A, you point your client at their AgentCard URL and it works.
2. **Cross-framework composition.** A LangGraph agent calling a Google ADK agent calling a BeeAI agent — they were built by different teams with different internals. HTTP alone doesn't make them interoperable. A2A does.
3. **Tooling.** Generic A2A clients, debuggers, traffic inspectors, observability tools — they exist because the protocol is uniform.
4. **Multi-agent orchestration.** If you're routing tasks across N agents based on capabilities, you need a uniform way to *describe* capabilities and *invoke* them. AgentCard + standard methods give you that.

Be honest with yourself, though: if you have one agent talking to one other agent, both built by your team, **A2A buys you almost nothing over a well-designed REST API**. You don't get the network effect. You're just paying the standardization tax without the payoff. A2A's value scales with the number of agent pairs and the heterogeneity of their implementations.

---

## What A2A Isn't

This is where things get interesting. A long list of capabilities people expect from "an agent protocol" are explicitly *not* part of A2A. The remainder of this article walks through each gap and the patterns for filling it.

### No stdio / local transport

A2A is HTTP-based today. There is **no stdio transport** in the current spec, despite this being one of MCP's most-used features.

This is a deliberate design difference from MCP. MCP supports two transports — stdio for local servers (host spawns a subprocess, JSON-RPC over stdin/stdout, logging on stderr) and Streamable HTTP for remote servers. A2A was designed for the opposite use case: connecting a "client agent" to a "remote agent" across organizational and infrastructural boundaries, where the remote might be a black box. It leans on web infrastructure precisely because HTTP(S) endpoints and JSON-RPC-style request/response messaging facilitate deployment across enterprise infrastructure and integration with existing gateways, auth layers, and observability.

That said, there's an open proposal to add stdio. GitHub issue #1074 on the `a2aproject/A2A` repo proposes introducing a new transport identified by the string "stdio" that allows communicating the A2A protocol over UNIX standard stdin and stdout streams. The design borrows directly from LSP: a simple message format identical to the Language Server Protocol message format — a header part followed by a message part. It's still under discussion, with open questions around subprocess ownership (SDK vs. host) and how to encode launch arguments in the AgentCard.

**Practical options for filling this gap:**

- Use MCP for local subprocess agents — that's exactly what it's built for.
- Run A2A endpoints on localhost over HTTP if you need A2A semantics locally. Works fine, just spins up a real socket.
- Track issue #1074 if you want stdio specifically in A2A.

### Governance and IAM: hooks only, no system

A2A handles authentication and authorization at the *transport* layer by piggybacking on HTTP standards. It's designed with security, authentication, and observability in mind, but the authentication credentials provisioning process is out of A2A scope and must be handled separately. The protocol leverages existing industry standards for securing HTTP communications.

What A2A *does* cover on the governance side:

- AgentCards declare what auth schemes an agent requires — typically OAuth 2.0, API keys, or OpenID Connect — and clients are expected to satisfy those before connecting.
- Agent discovery and capability advertisement: AgentCards are discovery documents at `/.well-known/agent-card.json` advertising identity, endpoint, capability flags, auth requirements, and a skills list. An agent can declare *who it is* and *what it can do* in a machine-readable way.

What A2A leaves to you (this is where it gets thin):

- **No standard agent identity model.** There's no defined way to say "this agent acts on behalf of user X with these scopes" — you have to bolt that on with your own auth layer.
- **AgentCard signing is optional, not enforced.** JWS signing exists in spec but isn't required. Unsigned cards can be spoofed. Discovery itself isn't trusted by default.
- **No typed skill contracts.** AgentCards can list skills with name, description, inputModes/outputModes, and examples — but that leaves ambiguity around required parameters and makes policy enforcement ("agent X may only call skill Y with parameter Z in range R") hard to do at the protocol level.
- **Governance of the protocol itself** is handled by the Linux Foundation, which is governance in the org sense but doesn't translate to runtime controls.

If you're thinking about agent IAM the way you'd think about user/service IAM — identity provider, scoped tokens, audit trails, policy decision points, delegation, on-behalf-of flows — A2A gives you the hooks but not the system. You'd typically pair it with an OAuth provider, a policy engine (OPA, Cedar), an API gateway for observability and rate limits, and your own conventions for agent identity (e.g., SPIFFE/SPIRE if you want workload identity, or DID-based schemes if you want decentralized identity — the ANP protocol goes that direction but A2A doesn't).

So: A2A is a *communication* standard with hooks for auth, not a governance or IAM framework. The protocol's own roadmap discussions acknowledge these gaps; whether they get filled in the spec or remain ecosystem concerns is still open.

### Access control propagation: not addressed

This is one of the genuinely unsolved problems in A2A, not just an "out of scope" choice. If agent A has access boundary X and agent B has access boundary Y, and A asks B to perform a task, what access boundary does B act within?

The short version: **B acts within B's own access boundary Y, using B's own credentials, with no standard way to know or constrain whose behalf it's acting on.**

Concretely:

- B advertises its auth requirements in its AgentCard.
- A must satisfy those requirements to call B — A presents credentials that authenticate *A to B*.
- B authenticates the *caller* (agent A), and authorizes the call based on what A is allowed to ask B to do.
- When B then goes off and does work — calling tools, hitting databases, invoking other agents — it does so as **B**, using **B's** credentials, within **B's** access boundary Y.

What A2A explicitly does **not** define:

- **No on-behalf-of semantics.** There's no standard field that says "A is calling B on behalf of user U with scopes S." The OAuth 2.0 token exchange spec (RFC 8693) exists for exactly this, but A2A doesn't mandate or even recommend a pattern.
- **No identity propagation.** B has no protocol-level way to know who *originally* initiated the chain. If a user asked A, who asked B, who asks C — by the time C is executing, the user's identity is gone unless someone passed it manually.
- **No scope intersection / least-privilege defaults.** If A has access to X and B has access to Y, A2A doesn't define how to compute the intersection and constrain B to act only within X ∩ Y for this specific call.
- **No standard delegation token format.** Each implementation invents its own.

#### The confused deputy problem

This setup is the classic "confused deputy" pattern, and it's a real security issue. Consider:

- Agent A has access boundary X (read-only customer data)
- Agent B has access boundary Y (read/write customer data + billing)
- User asks A to "summarize customer 123"
- A asks B to "get me data on customer 123"
- B has rights to do this *and more*. What stops B from also writing or billing?

Nothing, at the protocol level. B is trusted to behave correctly. If B is compromised, malicious, or just buggy, it can do anything within Y, not just what A's caller could have done directly.

Worse: if a user U could only access subset X' ⊂ X of data, and U asks A which asks B — B has no idea U exists. B will happily return data U was never authorized to see, and A might pass it back to U.

#### Patterns people actually use

Since the protocol punts, implementers have converged on a few patterns. None are standardized in A2A, but they're worth knowing:

1. **Pass-through tokens (token forwarding).** A receives a token from the user, A passes that *same* token to B. B acts as the user, not as itself. Simple, but requires B to trust A's token issuer, and requires the token to have audience B, not just A. Often broken in practice.

2. **Token exchange (RFC 8693).** A exchanges its user-scoped token for a new token at an identity provider, requesting a token valid for B's audience, optionally with reduced scopes. This is the "proper" way and what enterprise deployments tend toward. Requires an IdP that supports it (Okta, Auth0, Keycloak, Entra ID all do).

3. **On-behalf-of headers.** A sends its own service credentials to B, plus a separate header/claim asserting "acting on behalf of user U." B trusts A's identity, then applies authorization as if U were calling. Common in service meshes (mTLS + JWT claims). Requires B to trust A's assertion — which means A becomes a high-value target.

4. **Capability tokens / macaroons.** A receives a token that already encodes "may do X, Y for 5 minutes" and passes it (or a further-restricted derivative) to B. B's effective boundary is the *intersection* of its own permissions and the capability. Elegant for least-privilege; rarely deployed in agent systems today.

5. **Pre-issued scoped credentials per agent-to-agent edge.** Static per-relationship credentials: "A→B uses key K_AB which only permits the subset of Y that A is allowed to invoke." Configured ahead of time, no runtime exchange. Simple but doesn't handle per-user variation.

6. **Pure trust ("ambient authority").** B trusts that A wouldn't ask it to do anything wrong. Common in prototypes and internal systems where everything is behind the same VPC. Doesn't survive a security review.

#### What this means for your design

A few practical takeaways:

- **Don't assume A2A gives you delegation.** If you're building a system where user permissions matter, you have to design the auth flow yourself. Pick one of the patterns above and apply it consistently.
- **Treat B's blast radius as Y, always.** When you decide whether to let A call B, assume B will act with B's full authority. If that's unacceptable, either narrow Y (give B less access) or put a policy enforcement point in front of B that restricts what it'll do based on caller identity.
- **Be explicit about whose identity flows through.** Decide upfront: is this a *user-acting* chain (user's identity propagates, every agent acts as the user) or a *service-acting* chain (each agent uses its own service identity, user authorization checked only at the front door)? Mixing the two is where most real-world vulnerabilities live.
- **Audit at every hop.** Without identity propagation, your only forensic story is "agent A called agent B with payload P." If something goes wrong, you need to be able to reconstruct who *originally* asked. Log the original user identity at every agent boundary, even if the protocol doesn't carry it.
- **Watch out for prompt-injection-driven privilege escalation.** A's input might contain user-supplied text. If that text manipulates A into asking B for something the user couldn't have asked directly, you have a security problem that's invisible to A2A's auth layer. This is *the* big agent-security topic right now and A2A doesn't help with it.

#### Where the ecosystem is going

A few signs of where this might land:

- **AGNTCY** is explicitly trying to add identity propagation and policy enforcement on top of A2A.
- **OAuth working groups** are discussing agent-specific extensions to token exchange.
- **SPIFFE/SPIRE** is being adopted in some agent deployments for workload identity, with user identity layered on top via JWT.
- **OpenID Foundation** has early drafts of "AI agent identity" specs floating around.
- The A2A community has open issues on identity propagation, but no merged spec yet.

### Topology: unspecified

A2A is deliberately topology-agnostic. It defines how two agents talk, not how a system of agents should be arranged.

The protocol scope is narrow on purpose. It defines a client-server interaction between two agents, how to discover a remote agent, how to send a task and receive results, and the wire format. That's it. There's no concept in the spec of "the system," "the mesh," "the orchestrator role," or "the registry." Every interaction is a bilateral client→server call. If agent A calls agent B, A is the client for that exchange. If B later calls C, B is the client for *that* exchange. Roles are per-call, not assigned globally.

Because A2A only defines the edge between two agents, any topology you can build out of directed point-to-point calls is valid:

- **Star / hub-and-spoke** — one orchestrator agent calls many specialists. Most common pattern today (e.g., a "purchasing concierge" calling burger and pizza seller agents).
- **Pipeline / chain** — A calls B, B calls C, results flow back. The Oracle multi-agent RAG example (planner → researcher → reasoner → synthesizer) is this shape.
- **Mesh / peer-to-peer** — any agent can call any other. Nothing in A2A prevents this; it's just operationally harder because you need discovery.
- **Hierarchical** — orchestrators of orchestrators. Sub-agents inside one A2A endpoint are invisible to callers (the "opaque agent" principle), so hierarchy can exist at multiple levels.
- **Broadcast / pub-sub** — A2A doesn't support this natively. You'd build it on top with a fan-out orchestrator or external message bus.

The things you'd need to actually run a *system* of agents aren't in the spec: discovery beyond a single URL, routing and orchestration logic, identity and trust across the mesh, failure semantics across multiple hops, versioning and capability negotiation across the mesh. All of these are outside the protocol.

#### Where topology *does* get specified

If you want opinions about topology, you have to look one layer up:

- **Frameworks** — LangGraph encodes topology as a directed graph in code; Google ADK has its own orchestration model; AutoGen has conversation patterns (group chat, hierarchical). These sit *above* A2A.
- **AGNTCY** (Cisco's Internet of Agents) — adds discovery, group communication, and observability on top of A2A and MCP. This is the closest thing to a topology layer.
- **ANP** — leans toward open-mesh topology because it's built on decentralized IDs and JSON-LD graphs. The topology is "the open internet."
- **Service meshes** (Istio, Linkerd) — if you're deploying A2A endpoints as microservices, the actual network topology is whatever your mesh provides. A2A doesn't know or care.

A useful analogy: A2A is to agent systems what **HTTP** is to web applications. HTTP doesn't specify whether your app is a monolith, microservices, three-tier, or serverless — it just defines the request/response between two parties. Topology is an architecture decision you make on top.

### Initiation direction: client-initiated only

A2A is fundamentally request/response, with B unable to initiate a new task back to A. But there are some asynchronous mechanisms that complicate the picture, so it's worth being precise.

#### The core model

A2A defines a clear role split *per task*: client agent (the one who initiates by sending a task) and remote agent (the one who receives and executes). Within a single task, only the client can start things. B (the remote) cannot spontaneously send a task to A. There's no "B noticed something interesting and decided to tell A about it" primitive. The roles are asymmetric and assigned by who called whom.

#### What B *can* do during the task

Even though B can't *initiate* a new task, B has several ways to push information back to A while the task is live:

1. **Stream task updates (SSE).** B can send state changes (`working`, progress updates, intermediate messages) to A over Server-Sent Events. A is "listening" because A opened the connection. This is server-push *within* the request A already made.

2. **Push notifications (webhooks).** If A registered a webhook URL when submitting the task, B can POST updates to that URL as the task progresses. This *looks* like B initiating something, but it's not a new task — it's a callback to a URL A pre-authorized for this specific task. A is still the originator; B is just calling back.

3. **Ask for input mid-task (`input-required`).** B can transition the task to `input-required` state, effectively saying "I need more info to continue." A then sends a follow-up message *within the same task*. This is closer to a dialogue than pure request/response, but A is still the one driving — B is asking, A is answering.

4. **Produce artifacts.** B emits named outputs that A can fetch. Still within the task A initiated.

So during a task, the communication is bidirectional in *information flow*, but unidirectional in *control* — A controls when tasks start and end; B controls execution and can request input or push updates within that envelope.

#### What B genuinely cannot do

The hard limits:

- **B cannot start a new task targeting A.** If B wants to ask A something unrelated, there's no protocol path. B would have to become a *client* in a separate A2A relationship and call A's endpoint — and that requires B to have A's AgentCard, credentials for A, and so on. Possible, but it's a separate A→B relationship inverted into a B→A one, not a continuation.
- **B cannot push to A outside of an active task.** After a task completes, the channel is closed. B can't say "hey, that thing we worked on yesterday — here's an update."
- **B cannot broadcast.** No pub/sub primitive in the protocol.
- **B cannot escalate authority.** B can't ask A "please give me more permissions to finish this" beyond the `input-required` pattern (and even there, A would have to interpret and act on the request).

#### The "B as client" workaround

Nothing stops B from being *also* a client agent in its own right. If B has A's AgentCard and credentials, B can initiate its own task targeting A. This is how peer-to-peer A2A topologies work — each agent is both a server (for incoming tasks) and a client (for outgoing tasks).

So technically, in a peer-to-peer setup:

- A → B (task 1, A is client, B is remote)
- B → A (task 2, B is client, A is remote) — completely separate task, separate context, separate auth

From the protocol's view these are two unrelated bilateral exchanges. There's no notion of "B replying to A's earlier task with a new task of its own." If you want to correlate them, you do it in your application logic (e.g., reference task 1's ID in task 2's payload).

This is the closest A2A gets to bidirectional initiation, and it requires B to know A's endpoint (discovery), B to have credentials valid for A, and A to be willing to be a remote agent. In a hub-and-spoke topology where A is an orchestrator and B is a worker, B usually *doesn't* have A's AgentCard or auth, so this inversion isn't available. In a true peer-to-peer mesh, it is.

#### Practical implications

- **If you need B to notify A of something unrelated later** → design A as also being an A2A server, give B its AgentCard, and let B initiate a new task. Or use an out-of-band channel (message queue, webhook to A's app backend) that isn't A2A.
- **If you need event-driven agent communication** → A2A is the wrong layer. Pair it with an event bus. Agents subscribe to events on the bus, react by initiating A2A tasks.
- **If you need long-running passive monitoring** ("tell me when X happens") → model it as a long-running A2A task that stays in `working` state and streams updates when events occur. A starts it once, B keeps it alive indefinitely, B streams when relevant.
- **If you need symmetric peer agents** → set every agent up as both client and server, share AgentCards out-of-band, and accept that "conversations" are actually pairs of independent tasks correlated by your application logic.

#### Symmetric peers and the contextId problem

A natural question if you go down the peer-to-peer path: if A and B are peers, do A→B and B→A share the same `contextId`?

**No — and this is one of the places A2A's minimalism creates a real awkwardness for symmetric peer designs.**

`contextId` is scoped to a single client-server relationship. When A calls B, A (as client) either generates a `contextId` or lets B generate one on the first message. That `contextId` lives in **B's** task/context store, keyed for the **A→B** direction.

When B then calls A as a separate task, B is now the client, and a new `contextId` is generated. That `contextId` lives in **A's** task/context store, keyed for the **B→A** direction. These two `contextId`s are independent identifiers in independent stores. The protocol has no notion of "the conversation between A and B" as a single shared entity.

A few reasons this isn't just an oversight:

- **Storage ownership.** Each agent stores context for tasks where *it* is the remote. There's no shared store the protocol mandates, and no sync protocol between A's and B's stores.
- **Generation rules.** The first message in a task either omits `contextId` (letting the remote generate one) or includes one chosen by the client. There's no mechanism for B to say "use the same ID my A→B context used."
- **Authorization scoping.** `contextId`s implicitly establish a trust scope. Cross-mixing them between directions could create confused-deputy situations.
- **The opaque agent principle.** B's internal context state isn't supposed to be visible or addressable by A from the outside. If they shared `contextId`s, that boundary blurs.

If you want a unified "conversation" view, you need to build correlation yourself. Options:

1. **Application-level conversation ID.** Define your own `conversationId` and pass it in the message payload — not as A2A's `contextId`, but as part of the content or metadata. Both A and B carry it forward in every task they initiate, in either direction.
2. **Reference the other direction's task ID.** When B calls A in response to something from A→B, include "in reply to taskId X from contextId Y" in the message payload.
3. **External orchestrator holds the conversation state.** A third party (or one of the agents wearing a different hat) holds the "real" conversation log.
4. **Pick one direction as canonical.** In many peer designs, despite the symmetry of *capability*, the actual interactions follow a request-response pattern within a single direction. A asks B, B uses `input-required` to ask back, A answers — all within the A→B task.

If you're considering symmetric peer A2A and asking this question, **option 4 is usually the right design**. The need for B→A initiation often dissolves when you realize that `input-required` lets B stay in control of a long-running interaction while still pulling information from A.

### Statefulness and lifecycle: defined at the task level, not the agent level

#### Are A2A agents stateful?

**The agent itself: unspecified. The task: yes, explicitly stateful.**

This is an important distinction. A2A defines state at the **task** level, not the agent level. Whether the agent process is stateful, stateless, ephemeral, or long-running is entirely the implementer's choice — the protocol doesn't care.

What A2A *requires* to be stateful:

- **Tasks** have a defined lifecycle with state transitions.
- **Contexts** (via `contextId`) group multiple tasks into a conversation thread, so multi-turn interactions need *something* to remember prior turns.

How you implement that statefulness is up to you. Common patterns: stateless agent + external store (Redis, Postgres) keyed by `taskId` / `contextId`; stateful agent process holding state in memory; durable workflow engine (Temporal, Restate) backing the agent.

The protocol just says "when I send you a message with this `contextId`, behave as if you remember the prior turns." How you do that is your business — opacity again.

#### Task lifecycle

Tasks move through these states:

- `submitted` — task accepted, not started yet
- `working` — actively executing
- `input-required` — paused, waiting for client input
- `auth-required` — paused, waiting for authentication
- `completed` — terminal success
- `failed` — terminal failure
- `canceled` — terminal, client-initiated stop
- `rejected` — terminal, agent refused the task
- `unknown` — state can't be determined

The first four are non-terminal (task is "live"). The last five are terminal — the task is done, no further transitions. Each state change can be streamed to the client via SSE or pushed to a webhook. So from the client's perspective, the task has a clear, observable lifecycle even though the agent's internal execution is opaque.

#### What state persists after a task completes

This is where A2A is interesting — it specifies what's *exposed*, not what's *stored*.

**Exposed by the protocol** after a task reaches a terminal state:

- **The task itself** — its ID, final state, and history. A client can query a completed task by ID and get back its final state.
- **Artifacts** — named outputs produced during the task. These are the deliverables, addressable after completion.
- **Message history within the task** — the turn-by-turn exchange that led to the result.
- **The context** — if the `contextId` is reused in a future task, the agent is expected to behave as though it remembers prior tasks in that context.

**Not specified by the protocol:**

- **Retention policy.** A2A doesn't say how long any of this must be kept. Some implementations purge tasks after hours; others keep them indefinitely.
- **Internal agent state.** Whatever scratchpads, intermediate reasoning, tool call history, or working memory the agent used during the task — none of that is exposed or required to persist.
- **Cross-context memory.** Whether the agent remembers anything across different `contextId`s (true long-term memory) is entirely up to the implementation.
- **Audit / observability data.** A2A doesn't define a logging or audit format.

#### A mental model

Think of A2A's state model like HTTP + a job queue: the agent is like a web server (could be stateless, could hold state, the protocol doesn't say); the task is like a long-running job with a tracked lifecycle (you can poll it, stream updates, cancel it, fetch results); the context is like a session (a way to thread related jobs together); the artifact is like a job's output file (explicitly named, retrievable, the durable result).

The protocol gives you addressable identifiers (`taskId`, `contextId`, artifact names) and a state machine for tasks. Everything else — what's stored, for how long, where — is implementation.

#### Practical implications

- **Don't assume cross-agent memory.** If you call agent A in one task and agent B in another, B has no idea what A did unless you explicitly pass artifacts or context. State doesn't "leak" between agents.
- **Don't assume cross-context memory within one agent either.** Unless you reuse the `contextId`, treat each task as a fresh start.
- **Build your own durability layer if you need it.** For audit trails, replay, debugging across agents, or long-term memory — none of that comes for free. Pair A2A with a workflow engine, event log, or observability stack.
- **Artifacts are your friend.** They're the one piece of post-task state the protocol takes seriously. Design your agent boundaries so important outputs are emitted as artifacts rather than living only in the agent's internal state.
- **`input-required` doesn't pause forever.** Most implementations time out a task sitting in `input-required` after some interval.

### Shared memory: explicitly out of scope

A2A explicitly treats agents as **opaque**. The core design principle is that agents collaborate *without* exposing internal state. They should be able to collaborate on long-running tasks while operating without exposing their internal state, memory, or tools. That's a feature, not an omission — it's what lets you compose agents built on different frameworks by different vendors.

What A2A *does* define for state-like concerns:

- **`contextId`** — groups messages into a conversation thread (session-like). Two agents talking over multiple turns can maintain continuity, but this is conversation state for *that bilateral exchange*, not shared memory across the system.
- **`taskId`** — identifies a single unit of work within a context.
- **Artifacts** — structured outputs (files, JSON, text) that one agent produces and another consumes. This is the closest A2A gets to "shared state," but it's pass-by-value: explicit handoff of named deliverables, not a shared store.
- **Messages** — the turn-by-turn payloads exchanged within a task.

What's **not** in the spec:

- No shared memory store, blackboard, or working memory primitive.
- No vector store / RAG interface for cross-agent knowledge.
- No way for agent B to *read* agent A's internal scratchpad mid-task.
- No standard for episodic memory, long-term memory, or memory consolidation across agents.

If you want shared memory in an A2A system, you build it out-of-band — a shared vector DB, a Redis-backed scratchpad, an MCP server that both agents query, etc. The protocol doesn't help you and doesn't get in the way.

### Mid-flight steering: minimal

Here A2A has *some* mechanism but it's coarse and not really "steering" in the rich sense.

**What A2A does support:**

- **Streaming progress.** Long-running tasks can stream updates via SSE. The client *sees* what's happening as it happens.
- **Task state transitions.** Visibility into the lifecycle. The `input-required` state is the closest thing to mid-flight interaction — the agent can pause and ask the client for more input before continuing.
- **Cancellation.** A client can cancel an in-flight task. Hard stop, not steering.
- **Push notifications.** The server can push state updates to a webhook on the client.

**What A2A does *not* support:**

- **Interrupting and redirecting.** There's no spec primitive for "you're going down the wrong path, try this instead" while the agent is mid-execution. You can cancel and restart, but that's it.
- **Injecting new context mid-task.** No standard way to say "by the way, here's a new constraint" partway through.
- **Adjusting parameters/tools mid-flight.** No way to modify the agent's available tools, model, or system prompt during execution.
- **Speculative execution / branching.** No primitive for "explore both options and let me pick."
- **Inspecting reasoning to steer it.** Opacity cuts both ways — you can't peek at the agent's chain-of-thought to nudge it, because the protocol explicitly hides that.

The `input-required` state is the only collaborative mid-flight hook, and it's *agent-initiated* — the agent decides when it needs input. The client can't proactively interject.

#### Why these gaps exist

Both the shared-memory and mid-flight-steering omissions follow from A2A's foundational bet: **agents are black boxes that communicate through well-defined task envelopes.** Shared memory would violate opacity. Rich mid-flight steering would require exposing internal execution state.

This is the opposite philosophy from a framework like LangGraph, where you *own* the graph and can interrupt any node, modify state, and resume — because the agent isn't opaque to you, it's *yours*. A2A is designed for the case where the agent on the other end might be from a different company, running code you'll never see.

#### Practical implications

- **Shared memory across agents** → put it outside A2A. Common patterns: a shared MCP server (vector DB, KV store) that all agents query; an event bus (Kafka, NATS) for state propagation; a workflow engine (Temporal, Restate) that holds the durable state and orchestrates A2A calls.
- **Rich mid-flight steering** → keep that *inside* a single agent's boundary, where you control the execution. LangGraph's `interrupt`/`resume`, OpenAI Agents SDK's handoffs, or Temporal signals all give you this *within* a controlled agent. Then expose the assembled thing over A2A to the outside world.
- **Lightweight steering across A2A** → use the `input-required` pattern. Design your agent to checkpoint at decision points and ask the client to confirm direction. This is cooperative steering rather than imperative.

### Response shape: `Message` vs. `Task`

One genuinely thoughtful piece of the spec: when a client calls `message/send`, the agent has a choice about what to return. The choice signals what *kind* of work this is.

**Response = `Message`** → "I answered right here, we're done." The agent processed the input synchronously and the response *is* the answer. No task was created. No state machine spun up. No lifecycle to track. It's effectively a one-shot request/response.

Example: client asks "What's 2+2?" → agent returns a `Message` containing "4". End of story.

**Response = `Task`** → "I've started working on this; here's the handle to track it." The agent acknowledged the request and kicked off a stateful, trackable unit of work. The response contains a `Task` object with a `taskId`, an initial state (usually `submitted` or `working`), and the agent's promise to make progress on it. The client can now poll, stream, or be webhooked about updates.

Example: client asks "Research the entire history of A2A and write a 50-page report" → agent returns a `Task` with `taskId: t-abc123, state: working`.

#### Why this distinction exists

The protocol could have forced *everything* to be a task — every interaction creates a task, even a trivial one. That would be uniform but wasteful. Spinning up task state, allocating an ID, persisting it, transitioning it through states — all overkill for "what's 2+2?"

It could also have forced everything to be synchronous — no tasks, just blocking responses. But then long-running work would have to fake it with polling endpoints or invent its own async mechanism.

A2A picks the middle path: **let the agent decide per-request whether this is task-shaped or message-shaped work**. Cheap, instant operations stay cheap. Long-running operations get the full lifecycle machinery. The agent — not the client — makes the call about which mode is appropriate.

#### How the client handles this

The client has to be ready for either. Pseudocode:

```
response = agent.send(message)
if response is Message:
    # Done. Use the content directly.
    display(response.parts)
elif response is Task:
    # Long-running. Track it.
    if streaming:
        for update in agent.stream(response.taskId):
            handle(update)
    else:
        poll_until_terminal(response.taskId)
```

This is why A2A clients always need to handle the polymorphic response.

#### The decision rule (informal)

The spec doesn't dictate when to return which, but the natural rule is:

- **Can I answer in the time budget of a single HTTP request?** → return `Message`
- **Will this take long enough that the client benefits from a handle to track it?** → return `Task`

Common thresholds: sub-second computation, simple lookups, formatting → `Message`. Anything involving tool calls, LLM reasoning chains, external API calls with latency, multi-step work → usually `Task`. Anything that might need `input-required` partway through must be `Task` (you can't pause a `Message`). Anything that might emit artifacts must be `Task` (artifacts attach to tasks).

#### The subtle ambiguity

Two agents implementing the same logical capability might disagree on whether it's "task-worthy." Agent A might return `Message` for "summarize this paragraph" because it's fast. Agent B might return `Task` for the same request because B prefers uniform handling, or because B might call external tools, or because B wants to stream the summary as it generates. Both are valid. The client must be agnostic.

This is why robust A2A clients essentially always implement the task path and treat `Message` responses as a fast-path optimization.

There's a related nuance: if the client calls `message/stream` instead of `message/send`, the agent should respond with task-mode behavior — because streaming implies "give me updates over time," which requires a task to update. So `message/send` → either `Message` or `Task` is valid; `message/stream` → effectively forces `Task` mode.

### Discovery: thin

A2A defines only one discovery primitive: the **AgentCard**.

An AgentCard is a JSON document that describes an agent: name, description, version, the A2A endpoint URL, supported input/output modes, skills the agent advertises, authentication requirements, capability flags, and optional metadata. It's the agent's "business card" — everything you need to start calling it, in machine-readable form. The convention is to host this at the well-known URL `/.well-known/agent-card.json`.

So if A knows that B is reachable at `https://b.example.com`, A can fetch `https://b.example.com/.well-known/agent-card.json` and get everything needed to interact with B. That's the *only* discovery mechanism A2A formally defines.

#### The implicit assumption: you already know the URL

Notice what that mechanism *requires*: A already has to know B's base URL. The well-known path is a standard place to look *given* you know the host. But A2A says nothing about how A learns that B exists at `b.example.com` in the first place. This is the discovery gap. A2A defines "how to read an agent's card" but not "how to find agents in the wild."

#### How discovery actually happens in practice

Since the protocol punts, deployments have converged on a few patterns. None are A2A-standard; they're what people do.

1. **Static configuration.** By far the most common. A is configured (via env vars, config file, secrets manager) with B's URL. Pre-deployment, an operator decides which agents talk to which, and bakes the URLs in. Simple, works, doesn't scale to dozens of agents or dynamic environments.

2. **Agent registry / catalog.** A separate service holds a list of available agents. A queries the registry — "give me agents that can do X" — and gets back URLs and AgentCards. The registry might be a simple JSON file, a dedicated service, a directory in an orchestration framework (LangGraph, Google ADK, BeeAI all have variants), or a service mesh's service directory (Consul, Kubernetes services). A2A doesn't define a registry API. Every implementation invents its own.

3. **DNS-based discovery.** Some deployments use DNS SRV records or similar to locate A2A agents. Combine with the well-known path and you get "look up `_a2a._tcp.example.com`, find the host, fetch the AgentCard." Works well in enterprise environments with existing DNS infrastructure.

4. **Service mesh discovery.** If you're running on Kubernetes with Istio, Linkerd, or similar, agents have service identities and can find each other via the mesh's discovery layer. A2A endpoints are just services like any other.

5. **Out-of-band introduction.** A human (or another agent) hands one agent the URL of another. Common in orchestration scenarios — the orchestrator agent's config includes the URLs of its worker agents.

6. **Federation / referral.** Agent A asks agent B "do you know anyone who can do Y?" and B returns a URL or AgentCard for agent C. There's no A2A-standard way to do this — it's just a pattern.

7. **The "Internet of Agents" vision (largely aspirational).** The long-term vision: agents discover each other across the open internet, like how websites discover each other today. Standardized registries, searchable directories, cryptographic identity. AGNTCY and ANP are aiming at this. A2A alone doesn't get you there.

#### What A2A genuinely lacks

- **No capability search.** AgentCards describe skills, but there's no protocol for "find me an agent that can do X."
- **No federated lookup.** No equivalent of DNS for agents.
- **No trust model for discovery.** AgentCard signing exists but is optional and rarely used.
- **No versioning conventions for discovery.** If B updates its capabilities, how do clients learn? Not specified.
- **No agent metadata standards beyond the card.** Things like SLA, pricing, geographic location, data residency.

#### Practical recommendations

- **Decide your discovery model up front.** Static config is fine for prototypes. For anything larger, pick a registry pattern early.
- **Cache AgentCards with explicit invalidation.** Don't refetch on every call, but don't cache forever either. A few minutes to an hour is typical.
- **Treat AgentCard fetching as part of your auth flow.** Some auth requirements (e.g., OAuth audience values) are in the card.
- **Validate AgentCards.** Especially in less-trusted environments — check the schema, verify signatures if present, sanity-check the endpoint URL.
- **Don't build your system to require dynamic discovery if you don't need it.** Most production agent systems work fine with statically configured peers.

---

## Comparison to Adjacent Protocols

A2A doesn't exist in a vacuum. The agent protocol space has gotten crowded fast, and it's worth knowing where A2A sits.

### Direct alternatives (agent-to-agent communication)

**ACP (Agent Communication Protocol).** Originally from IBM's BeeAI team, now merged with A2A under the Linux Foundation but still referenced as a distinct approach. ACP introduces REST-native messaging via multi-part messages and asynchronous streaming to support multimodal agent responses. It was a REST-native performative messaging layer with async-first design and offline discovery. Since the merger, BeeAI uses A2A adapters, but ACP's design influence remains visible.

**ANP (Agent Network Protocol).** The decentralized alternative. ANP is a decentralized discovery and collaboration protocol built on decentralized identifiers (DIDs) and JSON-LD graphs for open-internet agent marketplaces. Where A2A assumes enterprise deployments behind known endpoints, ANP targets open-internet agent ecosystems where you don't know in advance who the agents are. Think "agents as web citizens" with cryptographic identity baked in. This is the protocol to look at if you care about the identity gaps in A2A.

### Adjacent / complementary protocols

**MCP (Model Context Protocol).** Anthropic's protocol, often discussed alongside A2A but solves a different problem. MCP is the emerging standard for connecting LLMs with data, resources, and tools. A2A is an application-level protocol that enables agents to collaborate in their natural modalities. The conventional wisdom is "MCP for tools, A2A for agents." MCP supports stdio (the thing A2A doesn't) and Streamable HTTP.

**AGNTCY (Cisco's Internet of Agents).** Not a competing wire protocol so much as a framework that builds on top. Agntcy is a framework that provides components to the Internet of Agents with discovery, group communication, identity and observability, and leverages A2A and MCP for agent communication and tool calling. So it's complementary — it uses A2A and adds the governance/identity layer A2A omits.

### Older / academic lineage

Worth knowing these exist even if you won't deploy them: **FIPA-ACL** (the classic agent communication language from the late 90s, performative-based with inform/request/propose/etc.), and **KQML** (even older, similar idea, mostly historical now).

### How to pick

A rough decision tree:

- **Agents within one org, behind known endpoints, enterprise auth** → A2A
- **Agents discovering each other on the open internet, no pre-shared trust** → ANP (or A2A + your own DID layer)
- **Agent talking to tools/data, not other agents** → MCP
- **You need governance, identity, and observability on top of A2A/MCP** → AGNTCY or roll your own
- **Local multi-agent system on one machine** → ACP patterns, or just MCP if it fits, or wait on the A2A stdio proposal

In practice most production stacks today end up dual-stack: MCP for tool access and A2A for cross-agent delegation, with a custom or AGNTCY-style governance layer wrapping both. The protocols aren't really competing — they're stratifying.

---

## LangChain / LangGraph and A2A

As of late 2025, LangGraph supports A2A natively for the **server side**: you can expose a LangGraph agent as an A2A endpoint by upgrading to `langgraph-api>=0.4.21` and deploying with a message-based state structure. The agent card includes the assistant's name, description, available skills, supported input/output modes, and the A2A endpoint URL for communication.

The integration handles the protocol semantics for you: the A2A protocol uses two identifiers to maintain conversational continuity — `contextId` (groups messages into a conversation thread, like a session ID) and `taskId` (identifies each individual request within that conversation). On the first message, omit both; the agent will generate and return them.

There's also a LangSmith tracing benefit: when multiple agents communicate over A2A, LangSmith can group all their traces into a single thread, giving you a unified view of the entire multi-agent conversation. The Agent Server A2A endpoint automatically converts the A2A `contextId` to `thread_id` for LangSmith tracing.

The shape of support is: **expose a LangGraph agent *as* an A2A server**. So other A2A-compatible agents (Google ADK, BeeAI, etc.) can call into your LangGraph agent over the standard protocol. The official docs include a Google ADK agent interacting with a LangChain agent over A2A as a worked example.

The reverse direction — treating remote A2A agents as first-class nodes inside a LangGraph graph, similar to how `langchain-mcp-adapters` lets you pull MCP tools in — is an open feature request, not yet native. LangGraph currently supports MCP adapters via langchain-mcp-adapters, which allow external tools to be integrated as `@tool`s. However, in many cases external systems are not just tools but full agents, and sub-graph integration for A2A isn't shipped yet.

Before the native endpoint landed, the main path was `python-a2a`, a community library that bridges both directions. It offers three main integration patterns: use LangChain agents as A2A agents, use A2A agents as tools within a LangChain workflow, or mix both. Still useful if you want bidirectional bridging today or if you're on an older LangGraph version.

**Quick decision guide:**

- **Want to expose a LangGraph agent over A2A** → use the native endpoint (`langgraph-api>=0.4.21`), straightforward.
- **Want to call remote A2A agents from inside a LangGraph workflow** → use `python-a2a` adapters, or wrap A2A calls manually as a node, until native sub-graph support ships.
- **Just experimenting** → the IBM/watsonx tutorials and the Oracle multi-agent RAG writeup are good end-to-end references.

---

## The Recurring Pattern

You may have noticed by now that the same pattern shows up in section after section. Across every gap above, A2A makes the same trade-off:

> **Specify the bilateral interaction envelope precisely. Leave system-level concerns to higher layers.**

This is the protocol's central design choice. It shows up in:

- **Transport** — HTTP only, with stdio still just a proposal.
- **Governance and IAM** — hooks for auth, no identity model.
- **Access control propagation** — define A→B auth, nothing about chained delegation.
- **Topology** — bilateral edges only, no system-level architecture.
- **Initiation direction** — client-initiated tasks, no reverse initiation or pub/sub.
- **Shared memory** — opacity by design, no shared store primitive.
- **Mid-flight steering** — `input-required` and that's it.
- **State persistence** — define what's exposed, not what's stored.
- **Discovery** — define the AgentCard format, not how to find agents.

Whether that's the right amount of protocol is genuinely debatable. The minimalism enables broad adoption across frameworks without dictating implementation choices — but it also means A2A alone won't get you to a production multi-agent system. You'll always need *something else* alongside it.

---

## Practical Takeaways

A few things to internalize if you're designing on top of A2A:

1. **A2A specifies the envelope, not the system.** Expect to build or adopt additional layers for everything from discovery to identity to memory.

2. **Don't over-invest in A2A for tightly coupled internal systems.** Two agents your team owns, behind one VPC, with one auth domain? Plain HTTP with a good API design is often the better engineering choice. A2A's value scales with the number of agent pairs and the heterogeneity of their implementations.

3. **Handle both `Message` and `Task` response shapes.** Don't assume one or the other. Different agents will choose differently, and the same agent may choose differently per request.

4. **Design auth flow explicitly.** A2A authenticates A to B; it doesn't propagate user identity or scopes. Pick a delegation pattern (token exchange, on-behalf-of headers, etc.) and apply it consistently. Audit at every hop. Log the original user identity at every agent boundary, even if the protocol doesn't carry it.

5. **Treat agents as opaque, and design your boundaries that way.** Anything important should cross the boundary as an artifact or message part, not as shared internal state. This is what makes cross-framework composition work.

6. **Use `input-required` instead of inventing reverse-initiation.** Most "B needs to ask A something" scenarios collapse to a cooperative pause in the existing task. Reach for true peer initiation only when genuinely needed.

7. **Don't assume cross-agent or cross-context memory.** Each task is its own context unless you explicitly thread them with `contextId`, and even then it's per-relationship. Artifacts are your durable handoff mechanism.

8. **Cache and validate AgentCards.** They're a security boundary as well as a discovery primitive.

9. **Pair A2A with the right complementary layers.** Workflow engines (Temporal, Restate) for durable orchestration. Event buses (Kafka, NATS) for event-driven flows. MCP for tool access. AGNTCY-style governance for identity and policy. Service mesh for transport-level concerns.

10. **Plan for the gaps to be filled by layers above A2A**, not by waiting for the spec to grow. The spec's design philosophy is to *stay* minimal. Most enrichment will come from frameworks, governance layers, and workflow engines wrapping A2A — not from A2A itself.

---

## Bottom Line

A2A is best understood as **"HTTP for agents, with a shared vocabulary."** It's a thin convention layer that gives heterogeneous agents a common way to find each other (sort of), talk to each other (precisely), and exchange work (with a defined lifecycle). It doesn't try to be a multi-agent operating system, and it shouldn't be evaluated as one.

For the right problem — composing opaque agents built by different teams across organizational boundaries — A2A's minimalism is exactly the right call. The whole point is that you *can't* assume shared infrastructure, shared identity, shared memory, or shared anything, because the agent on the other end might be from a different company running code you'll never see. Opacity isn't a limitation; it's the feature that makes interop possible.

For tightly coupled internal multi-agent products, A2A alone will leave you building most of the interesting infrastructure yourself, and you might be better served by a heavier framework that takes opinions on topology, memory, and orchestration. Or you'll end up using A2A *and* one of those frameworks, with A2A handling the external interface and the framework handling the internal one.

Either way, knowing what A2A *doesn't* do is at least as important as knowing what it does. The gaps are where most real systems get built, and the choices you make to fill them will define the shape of your multi-agent system far more than the protocol itself.