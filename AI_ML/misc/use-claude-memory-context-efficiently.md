# Using Context & Memory Efficiently in Claude Code

A practical guide to distributing knowledge across a team, loading it on demand,
keeping it compact, and knowing what belongs to *you* vs. what belongs to the *team*.

---

## 1. The core distinction: context vs. memory

Think **RAM vs. disk**.

| | **Context** | **Memory** |
|---|---|---|
| What it is | What's in the model's window *this turn* | Durable facts persisted to disk |
| Lifespan | Transient — rebuilt every turn, gone on `/clear` | Persistent — survives sessions |
| Size | Bounded (the window) | Unbounded (just files) |
| Role | The *working set* the model reasons over now | A *source* that selectively populates context |
| Analogy | Working memory / attention | Long-term memory |

**Context** is the union of everything assembled for the current turn: the
conversation so far, the loaded `CLAUDE.md` files, any recalled memory bodies,
files you've read, and tool output. It is rebuilt fresh each turn and discarded.

**Memory** is a persistent store that *feeds* context on demand. A memory file is
not "in context" until something makes it relevant. Most of memory is not in
context at any given moment — and that is the entire point.

> The line in one sentence: **memory is a candidate; context is what got selected.**
> `memory → (relevance match) → context`

```
        PERSISTENT SOURCES                         PER-TURN CONTEXT
  ┌───────────────────────────┐
  │ CLAUDE.md (always)        │ ─────always─────────┐
  │ memory/*.md (on relevance)│ ──if description───┐│
  │ skills (body on match)    │ ──if task matches─┐││
  └───────────────────────────┘                   ▼▼▼
                                          ┌─────────────────┐
  conversation history ──────────────────▶│  CONTEXT WINDOW │ ◀── file reads,
                                          │ (this turn only)│     tool output
                                          └─────────────────┘
```

`CLAUDE.md` is the instructive middle case: it is *persistent like memory* but
*always-loaded like the conversation*. It is effectively **memory you have pinned
to always be in context** — so keep it small.

---

## 2. Where context comes from (the layers)

When you send a prompt, Claude assembles context from layered sources:

1. **Instruction files (`CLAUDE.md`)** — the primary place for persistent instructions.
   - `./CLAUDE.md` (and parent dirs) — team-shared, committed to git.
   - `./CLAUDE.local.md` — personal project notes, gitignored.
   - `~/.claude/CLAUDE.md` — global, applies to all your projects.
   - Subdirectory `CLAUDE.md` — loaded on demand when files in that subtree are touched.
2. **System prompt + environment** — OS, shell, cwd, git status, model, tool defs.
3. **Skills, agents, hooks** — skill *descriptions* are always present; bodies load on use.
4. **Configuration** — `.claude/settings.json` (project), `settings.local.json` (personal), `~/.claude/settings.json` (global).
5. **Conversation + live reads** — history (summarized when long), plus anything read this turn.

**Precedence when sources conflict:**
1. Your explicit instructions (prompt, `CLAUDE.md`) — highest
2. Skills
3. Default system behavior — lowest

---

## 3. On-demand loading: don't load everything, always

The biggest efficiency lever is **not** stuffing everything into the root
`CLAUDE.md`. There are three loading models — pick the one that matches the
knowledge.

| Mechanism | When it loads | Use for |
|-----------|---------------|---------|
| **Nested `CLAUDE.md`** | Automatically, when a file in that folder's subtree is touched | Folder-specific conventions ("how code in *this* module works") |
| **Skills (`SKILL.md`)** | Only the *description* is in context; full body loads when the **task matches** | Task / workflow knowledge ("how to do X") |
| **Memory files** | Index always; a fact body loads when its `description` matches the task | Learned facts, recalled by relevance |
| **`@import` in CLAUDE.md** | **Eagerly**, whenever the parent loads | Shared snippets you always want alongside a given CLAUDE.md |

The same key drives all three: the **`description` / location** is the match
signal. On-demand loading is real distribution — nesting and skills — *not*
`@imports` stuffed into one root file (those are eager and always load).

### Example: distribute by folder (nested CLAUDE.md)

```
repo/
├── CLAUDE.md                      # always loaded: tiny, repo-wide rules + commands
└── src/
    ├── auth/
    │   ├── CLAUDE.md              # loads ONLY when working under auth/
    │   └── session.py
    ├── billing/
    │   ├── CLAUDE.md              # loads ONLY when working under billing/
    │   └── invoice.py
    └── api/
        └── CLAUDE.md
```

Working on an `auth` bug? The `auth/CLAUDE.md` auto-loads; `billing` and `api`
context never enters the window. The context stays lean and on-topic.

### Example: distribute by task (skill)

```markdown
---
name: add-endpoint
description: Use when adding a new REST endpoint to the service
---
# Adding an Endpoint
1. Register the route in the router module.
2. Add a request/response schema and validation.
3. Add a test fixture under tests/fixtures/.
4. Run the test suite for the affected module.
```

Only the one-line `description` sits in context until the task matches. Then the
steps load. Idle the rest of the time.

### Choosing

- Knowledge tied to a **place** in the tree → nested `CLAUDE.md`.
- Knowledge tied to an **action / workflow** → skill.
- **Always-needed and tiny** → root `CLAUDE.md`.
- Shared boilerplate for one `CLAUDE.md` → `@import` (remember: eager).

---

## 4. How memory loads at session start (and why it scales)

At the start of a new conversation, Claude Code loads **only the index**
(`MEMORY.md`) — one short line per memory, **no bodies**.

```
New conversation
   │
   ▼
MEMORY.md (index)        ← loaded in full, every session (one line per memory)
   │
   │  fact files stay on disk, NOT in context
   ▼
auth-token-ttl.md   user-email.md   billing-rounding.md   (bodies dormant)
```

As the conversation proceeds, a fact body is **recalled on relevance** — its
`description` matches the task, and the harness injects the body inside a
`<system-reminder>`. Facts whose descriptions don't match never enter the window.

**Recall quality = description quality.** The index line and the `description:`
are *all* the system sees when deciding whether to pull a fact. Write them
specific and keyword-rich.

```markdown
---
name: auth-token-ttl
description: Session tokens expire after 15 min; refresh logic must check exp before reuse
                ▲ this sentence is the match key — make it precise
---
```

---

## 5. Compaction & compression: keeping it lean

Two different things grow, and each has its own remedy.

### Conversation (short-term) — handled automatically
When the conversation grows large, Claude Code **summarizes** older turns
(compaction) and continues. You don't lose the thread, and you don't manage it.
`/clear` resets it entirely.

### Memory index (long-term) — your job to keep small
The one part that grows linearly with every memory is the **always-loaded
`MEMORY.md` index**. Keep it compact:

1. **One fact per file, tight descriptions.** Bodies are effectively free (they
   load on demand); only index lines cost you at startup.
2. **Lean on project-scoping (automatic).** The memory dir is keyed to the
   project path, so unrelated projects' memories never load here.
3. **Consolidate and prune regularly.** Merge duplicates, delete stale facts,
   trim the index. (e.g. a `/consolidate-memory` pass.)
4. **Don't store what's already in the repo.** Code structure, git history, past
   fixes, `CLAUDE.md` content — re-derivable, so they shouldn't take an index slot.

| | Loaded when | Cost | Remedy |
|---|---|---|---|
| `MEMORY.md` index | Every session, in full | Linear in # of memories | Prune / consolidate |
| A fact body | When its `description` matches | ~Zero until needed | Good descriptions |
| Conversation | The current session | Grows with turns | Auto-compaction; `/clear` |
| Global `~/.claude/CLAUDE.md` | Every session, every project | Always present | Keep small |

---

## 6. Personal vs. team-wide: the mental model

> **Memory = your private notebook. `CLAUDE.md` / skills = the team handbook
> (in git, reviewable, versioned).**

The memory dir is **not in the repo and not shared by default**: it lives under
`~/.claude/projects/...` (machine-local), is keyed to an absolute path (not
portable), and mixes personal facts (your email, your working style, corrections
you've given) with potentially team-relevant ones. Sharing it directly rarely
makes sense — it isn't portable and isn't reviewable.

The right channel for shared knowledge already exists: **commit it to the repo.**

| Want to SHARE with the team? | Put it here (committed to git) |
|---|---|
| Project conventions, architecture rules | `CLAUDE.md` (root + nested) |
| Workflows ("how to add an endpoint") | `.claude/skills/*/SKILL.md` |
| Specialized agents | `.claude/agents/*.md` |
| Permissions, hooks, env | `.claude/settings.json` |

| Keep PERSONAL (not shared) | Where |
|---|---|
| Learned facts, your preferences | `~/.claude/projects/.../memory/` |
| Personal project notes | `CLAUDE.local.md` (gitignored) |
| Personal overrides | `.claude/settings.local.json` |
| Cross-project personal rules | `~/.claude/CLAUDE.md` |

### The deciding question

```
Will I need this in a future conversation?
├── No  → leave it in the conversation, or a scratchpad note
└── Yes
     ├── Is it about ME (preference, style, my correction)?  → personal memory
     ├── Only in THIS project, just for me?                  → CLAUDE.local.md
     ├── A shared, reviewable project rule?                  → committed CLAUDE.md / skill
     └── True across ALL my projects?                        → ~/.claude/CLAUDE.md
```

### Promotion: the workflow that ties it together

1. You discover something during a session.
2. It lands in your **personal memory** (uncertain, or about you).
3. Once it proves to be a **durable team truth**, you *promote* it into a
   committed `CLAUDE.md` or skill — where the whole team (and their assistants)
   get it, and it goes through code review like any other change.

That promotion step **is** the personal/team line in practice:

- Stays personal / uncertain / about-you → **memory**.
- Becomes a shared, reviewable project rule → **`CLAUDE.md` / skill in the repo**.

---

## 7. Visibility

Everything is plain files you own:

- **Memory store** — `~/.claude/projects/<project>/memory/` (`MEMORY.md` index +
  `.md` fact files). `cat`, edit, `git`, or `rm` any of it.
- **Instructions** — `CLAUDE.md`, `CLAUDE.local.md`, `~/.claude/CLAUDE.md`.
- **What's loaded *this* session** — visible in the `<system-reminder>` blocks
  near the top of the conversation.

The *data* is fully transparent and yours. Only the *orchestration* (which facts
get recalled into a given turn, plus harness-injected extras) runs behind the
scenes — and it operates entirely on files you can inspect and change.

---

## TL;DR

- **Context = RAM (this turn). Memory = disk (persistent). `CLAUDE.md` = memory pinned always-on.**
- **Distribute** knowledge so it loads **on demand**: nested `CLAUDE.md` by folder, skills by task.
- At session start only the **memory index** loads; **bodies load by relevance** — so write sharp `description`s.
- **Compact** the long-term index (one fact/file, prune, consolidate); the conversation compacts itself.
- **Personal → memory / `*.local`.** **Team → committed `CLAUDE.md` / skills.** Promote facts up when they become shared truth.