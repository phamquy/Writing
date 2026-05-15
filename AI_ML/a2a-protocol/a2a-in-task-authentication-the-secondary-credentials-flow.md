---
icon: key
---

# A2A In-Task Authentication: The Secondary Credentials Flow

The A2A protocol spec contains this short passage about in-task authentication:

> **In-Task Authentication (Secondary Credentials):** If an agent needs additional credentials to access a different system or service during a task (for example, to use a specific tool on the user's behalf), the A2A server indicates to the client that more information is needed. The client is then responsible for obtaining these secondary credentials through a process outside of the A2A protocol itself (for example, an OAuth flow) and providing them back to the A2A server to continue the task.

It's one of the more practically important corners of A2A, and the spec is intentionally vague about the _mechanics_, which makes it hard to picture. This article walks through the flow concretely, then unpacks the design choices, edge cases, and what you actually have to build.

***

### The scenario the spec is describing

Before the flow, let's pin down when this even comes up.

Agent B is running a task that A submitted. Partway through, B realizes it needs to do something on the user's behalf that requires _separate_ credentials from the ones A used to call B. Common examples:

* B needs to read the user's Gmail to summarize emails — that requires the user's Google OAuth consent, which B doesn't have.
* B needs to post to the user's Slack workspace — needs Slack OAuth.
* B is a travel agent and needs to charge the user's Stripe-stored card — needs payment auth.
* B needs to query the user's GitHub repos via the GitHub API on the user's behalf.

In all of these, the credentials A used to authenticate to B are insufficient. A authenticated _itself_ to B (or possibly delegated the user's identity to B for B's own resources), but B now needs a _different_ token for a _different_ downstream system. Crucially, only the user can grant that consent — A can't just hand B the credential because A doesn't have it either.

This is what the spec means by "secondary credentials": creds B needs in-flight, that neither A nor B currently holds, that have to be obtained through some out-of-band auth ceremony.

***

### The high-level flow

In broad strokes:

1. A sends a task to B.
2. B starts working.
3. B hits the point where it needs the secondary credential.
4. B pauses the task and signals to A: "I need auth for X."
5. A relays this to the user (or its own auth machinery), runs an OAuth flow (or whatever), gets the credential.
6. A sends the credential back to B as a continuation of the same task.
7. B resumes work using the new credential.
8. Eventually B finishes and returns the result.

The A2A-defined parts are steps 4 and 6 — the signaling. The OAuth flow itself (step 5) is _explicitly_ outside the protocol, which is why the spec is brief about it.

***

### The concrete sequence

Let me walk through this in more detail with actual A2A primitives.

#### Step 1: A submits the task

```http
POST /a2a HTTP/1.1
Host: b.example.com
Authorization: Bearer <A's token to call B>
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "method": "message/send",
  "params": {
    "message": {
      "parts": [{"text": "Summarize my unread emails from the last 24 hours"}]
    }
  },
  "id": "req-1"
}
```

A authenticates to B per B's AgentCard requirements. The task is born.

#### Step 2: B starts working, returns a Task object

B knows this isn't instant, so it responds with a `Task` rather than a `Message`:

```json
{
  "jsonrpc": "2.0",
  "result": {
    "id": "task-789",
    "contextId": "ctx-456",
    "status": {"state": "working"},
    "history": [...]
  },
  "id": "req-1"
}
```

A now has a task handle and (likely) opens a streaming connection (`message/stream`) or registers a webhook to track progress.

#### Step 3: B realizes it needs Gmail access

B's internal logic determines it needs an OAuth token for Gmail with the `gmail.readonly` scope to fulfill this task. B doesn't have one for this user. So B transitions the task to `auth-required` state.

#### Step 4: B signals "I need auth"

This is the key moment. B updates the task's status:

```json
{
  "id": "task-789",
  "contextId": "ctx-456",
  "status": {
    "state": "auth-required",
    "message": {
      "parts": [{
        "kind": "data",
        "data": {
          "authRequest": {
            "service": "gmail",
            "scheme": "oauth2",
            "authorizationUrl": "https://accounts.google.com/o/oauth2/v2/auth",
            "tokenUrl": "https://oauth2.googleapis.com/token",
            "scopes": ["https://www.googleapis.com/auth/gmail.readonly"],
            "clientId": "<B's client_id for Google OAuth>",
            "audience": "https://gmail.googleapis.com/",
            "description": "Read access to your Gmail to summarize unread messages"
          }
        }
      }]
    }
  }
}
```

This gets pushed to A via SSE (if streaming) or via webhook. A few things to notice:

* The state is `auth-required`, which is a first-class A2A task state — A's client library should recognize and surface it.
* The _payload_ describing what auth is needed is **not standardized by A2A**. The spec leaves the shape open. Different agents and frameworks will do this differently — some use OAuth's own discovery documents, some define their own JSON schema, some return a simple `authorizationUrl` and expect the client to figure it out.
* B may include hints about what scopes it needs, what scheme (OAuth 2.0, OIDC, API key, mTLS cert, etc.), and human-readable descriptions for consent UIs.

#### Step 5: A obtains the credential (outside A2A)

This is where the spec waves its hands. What actually happens depends on who A is:

**If A is acting on behalf of a human user with a UI:** A's frontend redirects the user to the OAuth `authorizationUrl` (with appropriate `redirect_uri`, `state`, etc.). User logs into Google, sees the consent screen ("This app wants to read your Gmail"), approves. Google redirects back to A's callback URL with an auth code. A exchanges the code for an access token + refresh token. A now has the credential.

**If A is a server-side agent with pre-authorized credentials:** A looks up the user in its own credential store. If it already has a Gmail token for this user (cached from a prior interaction), it uses that. If not, it triggers the user's consent flow asynchronously (notification, email, etc.) and waits.

**If A is itself an agent without a user:** This gets philosophically weird. Without a human, there's no one to consent. A would either need pre-issued credentials (machine-to-machine OAuth client credentials grant), or it would need to fail the task because the secondary credential can't be obtained.

The point: **A2A doesn't care how this happens.** It just expects A to either come back with credentials or give up.

#### Step 6: A sends the credential back via `message/send`

A continues the existing task (same `taskId`, same `contextId`) with a follow-up message containing the credential:

```http
POST /a2a HTTP/1.1
Host: b.example.com
Authorization: Bearer <A's token to call B>
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "method": "message/send",
  "params": {
    "message": {
      "taskId": "task-789",
      "contextId": "ctx-456",
      "parts": [{
        "kind": "data",
        "data": {
          "authResponse": {
            "service": "gmail",
            "scheme": "oauth2",
            "accessToken": "ya29.a0Af...",
            "tokenType": "Bearer",
            "expiresIn": 3599,
            "refreshToken": "1//0g..."
          }
        }
      }]
    }
  },
  "id": "req-2"
}
```

Two important details:

* The message is bound to the existing `taskId` — this isn't a new task, it's a continuation. That's what lets B match it up with the paused work.
* The credential is in the message payload, not in an HTTP header. The HTTP `Authorization` header is still A's credential to call B; the _secondary_ credential travels inside the A2A message body.

There's a real security concern here, which I'll come back to.

#### Step 7: B receives the credential, resumes work

B's task handler sees the new message on `task-789`, recognizes it as an auth response, extracts the token, transitions the task back to `working`, and continues. From A's perspective (via stream or webhook), the task state goes back to `working` and eventually to `completed`.

#### Step 8: Final result

B finishes summarizing the emails, emits an artifact with the summary, transitions to `completed`. A retrieves the artifact and is done.

***

### A sequence diagram view

```
User      Agent A           Agent B          Google OAuth
 |          |                  |                   |
 |--ask---->|                  |                   |
 |          |--message/send--->|                   |
 |          |<--Task(working)--|                   |
 |          |                  |--starts working   |
 |          |                  |--needs Gmail      |
 |          |<--auth-required--|                   |
 |          |--need consent--->|                   |
 |<--OAuth redirect--|         |                   |
 |--login + consent------------|------------------>|
 |          |<--auth code------|-------------------|
 |          |--exchange code-->|------------------>|
 |          |<--access token---|-------------------|
 |          |--message/send (auth response)->|     |
 |          |<--Task(working)--|               |
 |          |                  |--calls Gmail-->|
 |          |                  |<--emails------|
 |          |                  |--summarizes   |
 |          |<--Task(completed)+ artifact------|
 |<--summary|                  |               |
```

***

### Variations and edge cases

Now for the interesting parts the spec doesn't spell out.

#### Who actually does the OAuth flow — A or B?

Two patterns exist, and the spec accommodates both:

**Pattern 1: A does the OAuth flow, passes the token to B.**

This is what I described above. A registers as an OAuth client with Google (or whatever), runs the flow, hands the resulting token to B. B uses it.

Pros: A owns the user relationship, A's app gets listed in Google's "third-party apps" panel. Cons: A must register OAuth clients for every downstream service any B might need. That doesn't scale.

**Pattern 2: B is the OAuth client, A just relays the redirect.**

Here B has registered as an OAuth client with Google. B sends A an `authorizationUrl` that points to Google with B's `client_id`. A's only job is to get the user to that URL, let them consent, and then capture the redirect back. The token is exchanged by B (or never seen by A — the OAuth flow might redirect directly back to B's callback URL, bypassing A).

Pros: B owns the integration with Google, A doesn't have to know anything about Gmail specifically. Cons: B becomes the user-facing OAuth app. The consent screen says "Agent B wants access to your Gmail" — even though the user is interacting with A. This is confusing UX.

There's also a **Pattern 3** where the OAuth callback redirects to A, A forwards the auth code to B, and B exchanges the code for the token. This minimizes who sees the access token but requires precise redirect orchestration.

The A2A spec doesn't pick one. The `authRequest` payload from B should be detailed enough that A knows which pattern is in play, but the actual conventions are framework-specific.

#### What if the user refuses or the OAuth flow fails?

A needs to tell B "no auth for you." Two options:

1. **A cancels the task.** Clean, but loses any partial work B has done.
2. **A sends back an "auth denied" message and lets B decide.** B might be able to fall back to a degraded path (e.g., "I can't read your Gmail but I can still give you generic email tips"), or B might fail the task itself.

The spec doesn't define a standard "auth denied" message shape — that's an implementation convention. Defensive A2A clients should at least support cancellation as a fallback.

#### What about token refresh?

Access tokens expire. If B's task runs longer than the token lifetime, B has a few options:

* **Use the refresh token directly** if A sent one. B refreshes silently. This requires B to have the OAuth client\_secret, which means it's really Pattern 2 above.
* **Re-enter `auth-required`** when the token expires, asking A for a refreshed token. A can refresh it (if A holds the refresh token) and send back. Works but adds round-trips for long-running tasks.
* **Expect short tasks.** Many B implementations just assume the token will outlive the task and fail otherwise. Crude but common.

For tasks that need refresh capability, A and B need to coordinate which side holds the refresh token, and that's not standardized.

#### What about multiple secondary credentials?

A task might need _several_ secondary auths — read Gmail, post to Slack, charge Stripe. B can re-enter `auth-required` multiple times during the same task, asking for each in turn. Or B can request a batch up front via a single `auth-required` with multiple service entries. Both are valid; the spec doesn't say.

In practice, asking for them one at a time is more user-friendly (consent fatigue from a big batch is real) but slower (more round-trips, more pauses).

#### What if A doesn't support the requested auth scheme?

A might not know how to do OAuth at all — maybe A is a simple script that only understands API keys. If B requests OAuth 2.0 and A can't do it, A has to fail gracefully. There's no standard "I don't support this scheme" response; A typically just cancels the task and surfaces an error to its user.

This is one reason agents tend to advertise their auth capabilities up front (in AgentCards) and orchestrators consider auth compatibility when choosing which B to call.

#### What about caching credentials across tasks?

If A has already obtained Gmail consent for the user (from a prior task), and B needs Gmail access again, does A have to ask the user again? No — A should cache the credential and reuse it. The `auth-required` flow only fires when the credential is missing or expired.

But: A and B have to coordinate the _scope_ match. If B's previous task needed `gmail.readonly` and the new task needs `gmail.send`, the cached credential is insufficient. Scope tracking is on A.

#### Where does the credential live during the task?

Once A sends the credential to B, B has it. For how long? A2A doesn't say. B might:

* Use it for this one task and discard immediately.
* Cache it for the lifetime of the `contextId`, so subsequent tasks in the same context don't re-prompt.
* Persist it long-term keyed by user ID, treating it as a stored grant.

Each choice has different security trade-offs. Persistence means more convenience but more risk if B is compromised. The spec is silent; this is an agent-implementer policy decision.

***

### The security elephant in the room

Sending OAuth access tokens in A2A message bodies has implications worth being honest about.

**The token traverses A2A's transport.** Which is HTTPS, so it's encrypted in transit. Good.

**But the token sits in A2A's task history.** Most A2A implementations log task messages. That message contains the token. So now the token is in B's task log, possibly indefinitely. If task history is queryable via `tasks/get`, the token might be retrievable by anyone with B's auth.

**Mitigations:**

* B should explicitly redact auth payloads from logged task history.
* B should treat secondary credentials as in-memory only, never persisted with the task.
* B should use the token, get the data, and then evict the token from the task's context.
* Some implementations encrypt the auth payload with a one-time key established at task creation, so even if the message body is logged, it's unreadable later.

None of this is in the A2A spec. It's the implementer's responsibility.

**A separate concern:** B might leak the token by including it in artifacts or messages B sends _back_ to A (or worse, to other agents in a chain). Defensive design: secondary credentials should be marked as "do not propagate" in the agent's internal handling.

**The on-behalf-of confusion** comes back here too. If A's user U has only `gmail.readonly` consent but B needs `gmail.send`, B can't escalate by requesting a broader scope unless the user re-consents. The user _will_ see Google's consent screen, so they can decline. But if A and B are colluding (or A is compromised), they could trick the user into consenting to broader scopes than they think they're granting. This is a general OAuth concern, not A2A-specific, but the agent layer makes it more salient because the consent screen is several abstraction layers removed from what the user thinks they're doing.

***

### What you actually have to build

If you're implementing the client side (A):

1. **Recognize `auth-required` task state.** Don't treat it as a generic error.
2. **Parse the `authRequest` payload.** Be prepared for non-standard shapes — different B implementations will encode this differently.
3. **Run the appropriate auth flow.** Have OAuth client setup for common providers (Google, Microsoft, Slack, GitHub, etc.) or a generic OAuth library that takes the discovery doc.
4. **Handle the redirect back to your app.** Capture the auth code, exchange for token.
5. **Send the credential back on the existing task** via `message/send` with the original `taskId`/`contextId`.
6. **Handle user refusal and OAuth errors gracefully.** Cancel the task with a clean error path.
7. **Cache credentials per (user, service, scope) tuple** to avoid re-prompting on every task.
8. **Refresh tokens proactively** if you're holding refresh tokens.

If you're implementing the server side (B):

1. **Detect when a secondary credential is needed.** This is task-specific logic — your code knows when it's about to call Gmail.
2. **Check if you already have a valid credential** for this user/scope (from earlier in the same task or context).
3. **Transition to `auth-required`** with a clear `authRequest` payload describing what you need.
4. **Wait for the continuation message.** Have a clean state machine for "paused task waiting for credential."
5. **Validate the credential** when it arrives (check scope, expiry, audience).
6. **Use it, then handle it carefully** — don't persist beyond what you need, redact from logs.
7. **Handle no-credential cases** — either fail the task or degrade gracefully.
8. **Plan for refresh** if your tasks might outlive token lifetime.

***

### Why the spec leaves so much undefined

You might be wondering why the spec is so vague here. A few reasons:

**The OAuth ecosystem is itself fragmented.** OAuth 2.0, OIDC, OAuth 2.1, PKCE, DPoP, mTLS, JWT bearer assertions, dynamic client registration — there's no single "OAuth flow." Mandating one would lock A2A out of half the auth schemes in use.

**Different deployments have different user-presence models.** A consumer-facing agent has a user with a browser. A B2B agent has a service identity. A scheduled job has neither. One mandated flow can't serve all of them.

**The auth obtaining itself doesn't need to be standardized for interop.** As long as A and B agree on "B says it needs X, A delivers X back via this task," the _how_ of getting X is between A and the user. So A2A only standardizes the signaling envelope, not the auth mechanism.

This is the same pattern we've seen across A2A — define the interaction shape precisely, leave the substantive work to layers above. Same minimalism, same trade-offs.

***

### TL;DR

The flow is:

1. A submits task → B starts working → B realizes it needs a secondary credential (e.g., OAuth token for a downstream service).
2. B pauses the task by transitioning to `auth-required` state and sends A a payload describing what it needs (provider, scopes, authorization URL).
3. A obtains the credential through a process A2A doesn't define — typically by walking the user through an OAuth consent flow, or by looking up cached/pre-issued creds.
4. A sends the credential back as a message on the same task (same `taskId`, same `contextId`), with the credential in the message body (typically as a `data` part).
5. B resumes work, uses the credential to call the downstream service, completes the task, and returns the result.

The protocol-defined parts are the `auth-required` state and the message exchange. The auth flow itself (OAuth dance, consent UI, token refresh) is explicitly outside the protocol — A2A treats it as a black box that A is responsible for. Real implementations have to handle a lot of nuance the spec doesn't address: which side runs the OAuth client, where the credential is stored, how it's refreshed, how to redact it from logs, how to handle multiple secondary auths in one task, and what happens when the user refuses.

This is one of those cases where A2A's minimalism cuts both ways: you get flexibility to plug in any auth scheme, but you also get responsibility for all the auth plumbing yourself, and you have to be careful about token handling because the protocol won't catch you if you mess it up.
