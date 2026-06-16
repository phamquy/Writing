---
icon: arrows-spin
---

# Loop Engineering: Designing the Agentic Loop

> *Working notes — June 2026. The capability of an AI agent is mostly a property of the loop, not the model. A single LLM call is stateless, context-bounded, and inert. Everything that makes an agent feel capable — that it can read your codebase, run your tests, notice it was wrong, and try again — lives in the loop you wrap around that call. This is the discipline of building that loop.*

---

## 1. From prompt engineering to loop engineering

In 2023 the craft was prompt engineering: how do you phrase a single request so one model call returns the best possible answer? It was a real skill, and it has a ceiling. A single call is a function — text in, text out — with no memory of the last call, no ability to look anything up, and no way to check whether its answer was any good.

The frontier moved. The interesting question now is not "how do I word this call?" but "what loop do I run this call inside of?" Take the same model — freeze the weights — and drop it into a naive loop versus a well-engineered one, and you get two systems that are barely recognizable as the same product. One forgets what it read three turns ago and confidently declares a broken build fixed. The other reads the failing test, edits the file, reruns the test, sees red, and keeps going until it sees green.

That delta is not the model. It is the engineering around the model. Call it **loop engineering**: the design of the cycle that turns a stateless next-token predictor into something that can carry out work.

This article does three things. It defines the loop from first principles. It enumerates the components you actually tune — the surfaces where loop engineering happens. And it gets concrete: we dissect a mature production loop (Claude Code), then build one, first by hand and then with Claude Code as the co-builder.

## 2. What the agentic loop actually is

Strip away the framing and an agent is a small state machine:

```text
gather context → model reasons → act (tool calls) → observe results → repeat → terminate
```

Each iteration takes the accumulated state, asks the model what to do next, executes whatever it asked for, folds the result back into the state, and goes around again — until some condition says stop.

The loop is not a decoration. It exists because a raw LLM call has three hard limitations, and each turn of the loop is a workaround for one of them:

1. **It is stateless.** The model retains nothing between calls. Continuity is an illusion the loop maintains by replaying history into every call. *Memory is something the loop does, not something the model has.*
2. **It is context-bounded.** The window is finite, so you cannot just shove everything in. Every turn, the loop makes an editorial decision about what the model is allowed to see.
3. **It is inert.** The model cannot run code, read a file, or call an API. It can only emit text. Tools plus the loop are what give it hands; the text it emits becomes an *instruction* the loop carries out.

This is why the right mental model is a state machine and not a chat transcript. A transcript is a record of things that were said. A loop is a controller: tool results are inputs, the model's output is a transition function, and the accumulated context is the state. Thinking of it as "a conversation" is the single most common source of bad agent design, because it hides every decision that actually matters.

## 3. The components of loop engineering

The loop is not one knob. It is a handful of independent engineering surfaces, each of which you design and tune separately. For each one, ask the same three questions: *what does it do, how do I control it, and how does it fail?*

**Context assembly — what enters the model each turn.** The system prompt, the running history, anything retrieved or freshly read. The core tension is relevance versus window budget: you want the model to see exactly what it needs and nothing else. Too little and it flies blind; too much and the signal drowns and every call gets more expensive. This is the surface most teams under-invest in and the one with the highest leverage.

**The action space — the tools.** Tool definitions, their input schemas, and the shape of what they return. Tools are the loop's API to the world, and their ergonomics bound what the agent can accomplish. A tool that returns a clean, structured error teaches the model how to recover; a tool that returns a 4,000-line stack trace poisons the context and the next three turns with it. Tool design is interface design, and the model is the user.

**Orchestration — the control flow.** This is the literal `while` loop: turn-taking, the iteration cap, the stop conditions, and the policy for when to hand control back to a human. Sub-agents are just nested loops with their own state. Most of what people call "the agent" is a few dozen lines here.

**Observation and verification — how the loop knows it is making progress.** Test runs, command exit codes, file diffs, assertions against the world. Without grounding, the loop has no feedback signal and will happily hallucinate success. The discipline is simple to state and hard to hold: *every claim of progress must be backed by an observation, not by the model's say-so.*

**Memory and state.** Short-term memory is this run's history. Long-term memory is anything persisted across runs — a scratchpad file, a vector store, a project's conventions doc. When history outgrows the window you need compaction: summarize the old turns into a compact form and carry the summary forward instead of the raw transcript.

**Termination and human-in-the-loop.** Done-detection, approval gates, permissioning. The line between "autonomous" and "supervised" is not a model setting — it is a loop policy. The same model and tools become an L2 co-pilot or an L4 autonomous worker depending entirely on where you put the stop states and approval prompts.

**Error handling and recovery.** Retries, backoff, and — most importantly — feeding errors back into the loop as observations rather than crashing on them. A well-built loop treats a failed tool call as just another input to reason about. A badly built one either dies or spirals, repeating the same failing action forever.

## 4. Anatomy walkthrough: Claude Code as a reference loop

Abstractions get slippery, so map the components above onto a real loop that ships to real users. Claude Code is a coding agent, and underneath the UI it is exactly the state machine from §2.

| Component (§3) | How Claude Code implements it |
| --- | --- |
| Context assembly | A `CLAUDE.md` conventions file, on-demand file reads, the running history, and automatic context compaction when the window fills. |
| Action space | `Read`, `Edit`, `Write`, `Bash`, `Grep`, `Glob` — each a tool with a typed schema and structured results. |
| Orchestration | The agent turn loop; sub-agents spawned as nested loops for isolated tasks. |
| Verification | Running the project's tests and commands via `Bash`, reading the actual output and diffs back into context. |
| Termination / HITL | Permission modes and approval prompts; "ask the user" is itself a terminal state of the loop. |
| Recovery | Tool errors (a failing command, a bad edit) are returned to the model as observations to reason about, not fatal exceptions. |

Watch it run one task — "fix this failing test" — and the loop becomes legible:

- **Turn 1.** *Gather:* read the test file and the code under test. *Reason:* form a hypothesis about the bug. *Act:* `Read` the relevant source. *Observe:* the file contents enter context.
- **Turn 2.** *Reason:* the off-by-one is on line 42. *Act:* `Edit` the file. *Observe:* the edit succeeds.
- **Turn 3.** *Act:* `Bash` runs the test suite. *Observe:* still red — a second assertion fails. This is the moment that separates a loop from a single call: the failure is not the end, it is the next turn's input.
- **Turn 4.** *Reason:* informed by the new error. *Act:* a second `Edit`, then rerun. *Observe:* green.
- **Terminate.** Done-detection fires — the goal is satisfied — and the loop hands back to the human.

Nothing here is exotic. That is the point. A production coding agent is the §2 cycle plus careful attention to each §3 surface, and the quality you feel using it is almost entirely in that attention.

## 5. Build it yourself: a minimal loop, then a real one

Reading about the loop only gets you so far. Build one and every component above stops being a word and becomes a line of code you wrote.

### 5.1 A minimal loop, by hand

Here is the entire thing — a `while` loop, one model call, two tools — in about forty lines of Python:

```python
import json, subprocess
from anthropic import Anthropic

client = Anthropic()

TOOLS = [
    {"name": "read_file", "description": "Read a file from disk",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}},
                      "required": ["path"]}},
    {"name": "run_shell", "description": "Run a shell command and return its output",
     "input_schema": {"type": "object", "properties": {"cmd": {"type": "string"}},
                      "required": ["cmd"]}},
]

def dispatch(name, args):                       # the action space
    if name == "read_file":
        return open(args["path"]).read()
    if name == "run_shell":
        return subprocess.run(args["cmd"], shell=True,
                              capture_output=True, text=True).stdout

def agent(goal, max_turns=15):
    history = [{"role": "user", "content": goal}]   # short-term memory
    for _ in range(max_turns):                      # orchestration + iteration cap
        resp = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=2048,
            system="You are a coding agent. Use tools, verify your work, then stop.",
            tools=TOOLS,
            messages=history,
        )
        history.append({"role": "assistant", "content": resp.content})

        if resp.stop_reason != "tool_use":          # termination
            return resp.content[-1].text

        results = []
        for block in resp.content:                  # act → observe
            if block.type == "tool_use":
                try:
                    out = dispatch(block.name, block.input)
                except Exception as e:
                    out = f"ERROR: {e}"             # recovery: error as observation
                results.append({"type": "tool_result",
                                "tool_use_id": block.id, "content": str(out)})
        history.append({"role": "user", "content": results})

    return "Hit max turns without finishing."       # a stop condition that matters
```

Trace it against §3 and the whole discipline is in front of you. The `system` string and `history` list are **context assembly**. `TOOLS` plus `dispatch` are the **action space**. The `for` loop with `max_turns` is **orchestration**, and the `max_turns` fallback is a **stop condition** that keeps a confused agent from running forever. Appending `tool_result` back into `history` is **observation**. The `try/except` that turns an exception into `"ERROR: ..."` is **recovery** — the agent gets to see what went wrong and reason about it. The whole notion of "verify your work, then stop" is **termination** driven by the model's own judgment, bounded by your cap.

It is a toy — no compaction, no approval gates, one weak stop condition — and it will still surprise you by reading a file, running a command, reacting to the output, and trying again. That surprise is the loop doing the work the model alone cannot.

### 5.2 A real one, with Claude Code as the co-builder

Now flip the perspective. Use Claude Code to extend that toy into something real: add a `write_file` tool, add a verification step that runs the test suite and refuses to terminate while it is red, add a `MEMORY.md` scratchpad the agent reads at the start of each run and updates at the end.

The point of doing it this way is not speed. It is that you are now operating one loop to build another, and you can feel the layering directly. You — the engineer — are the outer loop: you state a goal, Claude Code acts, you read the diff, you correct course, you approve. Claude Code is the inner loop, running its own gather-reason-act-observe cycle against your repo. The human-in-the-loop dev workflow and the agent's runtime loop are the same shape at two scales. Once you see that, "agentic" stops being a marketing word and becomes a structural observation: it is loops, nested, all the way down.

## 6. Measuring loop quality: cost and competence

You cannot tune what you do not measure, and the metrics that matter for a loop are not the metrics you would track for a single call. Per-call numbers — latency, tokens for this request, quality of this one output — are real but generic. The interesting properties of a loop are **emergent and cumulative**: they exist only across the whole trajectory.

**The cost axis.** Measure tokens — and dollars — **per completed task**, not per call. This is the trap that catches teams who only watch per-call cost: a loop can be cheap on every individual turn and still be ruinous, because each turn replays the accumulated history, so cost grows super-linearly with the number of turns. A thirty-turn run is not thirty times one call; it is closer to the sum of an arithmetic series. Track context utilization too — how much of each window was load-bearing versus bloat — and never forget to count the cost of **failed** runs, because an agent that burns 200K tokens and gives up still sent you the bill.

**The competence axis.** Because the loop decides its own "done," quality is not visible in any single output. You have to measure it at the loop level:

- **Task success rate** against a fixed, representative eval set — the only number that actually tells you whether the thing works.
- **Turns-to-completion** — how many iterations a task takes. Rising turn counts are an early warning that something upstream (context, tools, prompts) has degraded.
- **Trajectory quality** — did it take a clean path or thrash? Wasted tool calls, repeated actions, and backtracking are all measurable, and all signs of a loop that is confused rather than wrong.

**The frontier.** Put those two axes together and the real optimization target of loop engineering comes into view: the **cost-versus-quality frontier**. Every change you make either moves you along that curve (cheaper at the same success rate, or more capable at the same cost) or pushes the whole curve outward. That framing simply does not exist for a single call, and it is the right way to reason about whether a change was an improvement.

One distinction is worth stating plainly, because the same word — "eval" — gets used for both. The verification from §3 is **in-loop grounding**: it answers "does *this run* know it succeeded?" and it runs inside the loop. Measurement here is **out-of-loop evaluation**: it answers "is the loop *good across many runs?*" and it runs in your test harness, offline, over many tasks. Conflating them is how people end up with an agent that always *thinks* it succeeded and a team that has no idea how often it actually does.

Practically: log every turn — inputs, tool calls, token counts, outcome. Assemble a small eval set of representative tasks with known-good outcomes. Score runs against it, and watch for regressions every time you change the loop. *A loop you cannot replay and score is a loop you cannot improve.*

## 7. Failure modes and tuning levers

This is where the measurement pays off. Each common failure maps back to a §3 component, and you fix it by moving a number from §6 — not by guessing.

- **Non-termination / runaway iterations.** The loop never decides it is done, or thrashes until it hits the cap. *Lever:* tighter stop conditions, a sane iteration cap, and a clearer definition of "done" in the system prompt. *Watch:* turns-to-completion.
- **Context bloat and drift.** History grows until signal drowns and cost balloons. *Lever:* compaction, tighter context budgeting, and scoping what the agent reads instead of letting it slurp whole directories. *Watch:* context utilization, tokens-per-task.
- **Tool-error spirals and doom-loops.** The agent repeats a failing action forever. *Lever:* better structured error messages from tools, backoff, and an escalate-to-human path after N failures. *Watch:* trajectory quality.
- **Verification gaps.** The agent declares success and nothing actually works. *Lever:* ground every claim in an observation — run the test, read the output — and never let the model's self-report be the stop condition. *Watch:* task success rate, and the gap between in-loop and out-of-loop eval.
- **Over-permissioned action.** The loop does something destructive because nothing stopped it. *Lever:* human-in-the-loop gates on risky actions and least-privilege tools.

The meta-lever, and the one most worth internalizing: **invest in context, tools, and verification before you reach for a bigger model.** A model upgrade is the expensive, low-information move. Most of the time the loop is the bottleneck, and the loop is the part you control.

## 8. Principles and takeaways

- **Loop engineering beats prompt engineering.** The model is one component, and an increasingly replaceable one. The loop is where your leverage is.
- **The three highest-leverage surfaces are context, tools, and verification.** Get those right and a mid-tier model in a good loop will outperform a frontier model in a bad one.
- **"Agentic" is a spectrum set by loop policy,** not a property of the model. Autonomy, termination, and permissioning are decisions you encode in the control flow.
- **Measure across the trajectory, not the call.** Cost-per-task and success-rate over an eval set are the numbers that tell you the truth.

The model gets the headlines. The loop does the work — and the loop is the part you get to engineer.

## 9. Further reading and a challenge

- Context engineering — the discipline behind §3's first surface.
- Tool-use / function-calling specifications — how the action space is defined in practice.
- ReAct (reason + act) — the paper that named the reason/act interleaving at the heart of §2.
- Agent harness design and eval frameworks — the production-grade version of §6.

**Try this.** Take the forty-line loop from §5.1 and add one new capability — a `write_file` tool, a compaction step when history crosses a token threshold, or an approval gate before any shell command that mutates state. Then build a five-task eval set and measure whether your change moved the cost-versus-quality frontier. You will learn more from that one exercise than from re-reading this article.
