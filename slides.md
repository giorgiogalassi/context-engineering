---
theme: default
title: "Context Engineering: Stateless by Default, Stateful by Design"
author: Giorgio Galassi
highlighter: shiki
lineNumbers: false
drawings:
  persist: false
transition: slide-left
mdc: true
fonts:
  sans: Roboto
  mono: Roboto Mono
---

# Context Engineering

## Stateless by Default, Stateful by Design

Giorgio Galassi

<!--
Speaker notes:
Welcome the room. No preamble needed.
Let the title sit for a second before moving on.
-->

---

# You open your coding agent.

You worked on a feature yesterday.
You closed the session.

Today you open it again.

It has no idea who you are.

<!--
Speaker notes:
Don't explain yet. Just say this and pause.
Everyone in this room has been here.
Not a dramatic war story. Just Tuesday.
The agent asks what the project does.
You explain it again.
It asks about the conventions.
You explain those again.
Three months in, you're still explaining the same things.
This is the problem.

"How many of you have explained the same thing to an AI agent more than once this week?"
-->

---

# Three ways context fails

1. **<span class="accent">Session amnesia</span>** — everything resets when the session closes
2. **<span class="accent">Cross-project blindness</span>** — decisions made here never inform the next feature
3. **<span class="accent">Tool switch loss</span>** — context built in one tool is invisible to another

<!--
Speaker notes:
These are not edge cases. They compound.
Start with the ones everyone feels immediately.
Session amnesia is the daily tax. You pay it every morning.
You opened the session yesterday, did good work, closed it.
Today you start again and explain the same things.
Cross-project blindness is the silent one.
You solved this architectural problem six months ago on a different project.
You don't remember. Neither does the agent. You solve it again.
Tool switch loss comes third deliberately — it's the growing problem,
but it requires thinking about multiple tools which not everyone does yet.
Save it as the bridge: "and it gets worse when you use more than one tool."
This sets up the larger scale section later without jumping ahead.
-->

---

# Prompt engineering vs Context engineering

**Prompt engineering**
What you say to the agent.

**Context engineering**
What the agent already knows before you say anything.

<!--
Speaker notes:
This distinction is the reframe everything else depends on.
Prompt engineering gets a lot of attention. It's visible, it's immediate, it feels like leverage.
Context engineering is invisible when it works. You don't notice it.
But it's where the real compounding happens.
In my experience, it's the higher-leverage skill when building with AI agents.
Not because prompts don't matter. They do.
But because context is upstream of every prompt you'll ever write.
-->

---

# Context engineering

The discipline of deciding:

- **<span class="accent">what</span>** your agents know
- **<span class="accent">when</span>** they know it
- **<span class="accent">how</span>** that knowledge survives session boundaries and tool switches

<!--
Speaker notes:
This is the working definition we'll use for the rest of the talk.
Simple. Three dimensions.
We'll cover all three through the techniques.
-->

---

# The instinct: fix it with a better prompt

_"Here's what we discussed. Here's what we decided. Please continue."_

It works once, but it doesn't scale. Every session you're rebuilding context that should have been persisted.

<!--
Speaker notes:
Everyone in this room has done this. Including me.
The problem with this approach is that it puts the burden on you,
every single time, to reconstruct context that should have been persisted.
You become the memory. That's the wrong design.
The problem isn't what you say at the handoff.
It's that nothing was persisted before the handoff happened.
-->

---

# What is context?

Think of the model as a CPU.

The context window is RAM: everything the model can see and reason about right now.

It has a hard size limit, resets between sessions, and degrades in quality as it fills up — often well before hitting that limit.

<!--
Speaker notes:
Before we get to the techniques, we need to agree on what we mean by context.
The CPU/RAM analogy is the clearest mental model available.
The CPU (the model) doesn't change. Its capability is fixed.
What changes is what you put in RAM.
Three things worth saying explicitly:
Finite: every model has a hard token limit. You can't add more RAM.
Resets: every new session starts empty. Nothing carries over by default.
Degrades: this is the subtle one. It's not just that context runs out.
Performance degrades continuously as context fills, even well below the limit.
A model with a 200k token window may start showing degradation at 50k tokens.
The decline is a gradient, not a cliff.
This is why context engineering matters. You're not just managing space.
You're managing attention.
-->

---

# What competes for that space

| Layer                | What it contains                             |
| -------------------- | -------------------------------------------- |
| System prompt        | Agent identity, rules, control flow          |
| Tool definitions     | Schemas for every tool the agent can call    |
| Tool results         | Outputs from previous tool calls             |
| Retrieved knowledge  | Documents, search results, RAG chunks        |
| Conversation history | Everything said so far, every turn           |
| Memory               | Facts and preferences from previous sessions |
| Agent state          | Current plan, progress markers, scratchpad   |

Left unmanaged, each of these layers contributes to a known failure mode.

<!--
Speaker notes:
Seven categories, all competing for the same finite space.
Walk through them briefly, don't dwell.
The important point is that most of this content arrives without you asking for it.
Tool results accumulate automatically. Conversation history grows linearly.
Retrieved documents can be thousands of tokens each.
And none of it leaves unless you explicitly design it to.
This is the default: context fills up with everything that happened,
whether it's still relevant or not.

One specific beat worth hitting:
Tool definitions alone, if you have many MCP servers connected,
can consume thousands of tokens before any work has started.
That's the cost you pay on every single turn, for tools you may never use.
Researchers call this context confusion: too many tool definitions cause the model
to call the wrong tools or produce lower quality outputs.
One study found a model failed completely with 46 tools available,
and worked fine when trimmed to 19. The context wasn't too long.
It was too crowded.
-->

---

# When context isn't managed

- **<span class="accent">Context confusion</span>** — too many tools or irrelevant definitions cause wrong tool calls
- **<span class="accent">Context distraction</span>** — accumulated history causes the model to echo noise instead of reason
- **<span class="accent">Context poisoning</span>** — stale *or hostile* content enters once, then is re-read every session after
- **<span class="accent">Context clash</span>** — contradictory information produces inconsistent behavior

<div class="callout">

Three of these are hygiene. <span class="accent">Poisoning is a security problem</span> — because the output of agent A is the input of agent B.

</div>

<!--
Speaker notes:
These are the four named failure modes from the research community.
They map directly to what we just saw in the table.
Context confusion: tool definitions eating thousands of tokens before any work starts.
The model sees 46 tools and starts calling the wrong ones.
One study: a model failed completely with 46 tools, worked fine with 19.
The context wasn't too long. It was too crowded.
Context distraction: conversation history and tool results accumulating over a long session.
The model stops reasoning from first principles and starts echoing recent content.
Context poisoning: stale decisions, outdated architecture notes, superseded requirements
all sitting in context alongside current work. The agent builds on bad foundations.
Context clash: the system prompt says one thing, a retrieved document says another.
The agent can't reconcile it and produces inconsistent output.
The next two slides show what context distraction and context poisoning
look like in practice, with real measurements.
-->

---

# Lost in the middle

Models don't degrade uniformly. They show a **U-shaped attention curve**: best recall at the start and end of context, worst in the middle.

Liu et al., **2023**: information buried in the middle dropped accuracy by up to **<span class="accent">30 percentage points</span>** — on that year's models and tasks.

The curve has flattened since. It has not gone away.

![Lost in the Middle — U-shaped attention curve](/src/lost-in-middle.png)

<!--
Speaker notes:
This is the empirical proof that the degradation point is not theoretical.
The original research tested multi-document question answering: same question,
same documents, but the position of the relevant document varied.
Performance was highest when the answer was at the start or end of the context.
When it was in the middle, accuracy dropped by 30+ percentage points.
Think about what this means for a session that starts with clear instructions
and then accumulates 50 tool calls worth of output. Those original instructions
are now buried in the middle. They effectively disappear.
This is why index-first loading matters: critical orientation information
always stays at the top, not buried under accumulated noise.
Nuance worth knowing for Q&A: frontier models like Gemini 2.5 Flash are
improving on this. Frame it as a design principle, not an unsolved crisis:
bounded, structured context is cheaper and more predictable regardless.
-->

---

# Context rot

Performance doesn't just degrade when context is full. It degrades as it fills — gradually, continuously, long before hitting any limit.

Chroma, **July 2025** — 18 frontier models. <span class="accent">Every single one got worse</span> as input length increased.

The effect is structural. The specific numbers are not.

![Context Rot — performance degradation across 18 models (Chroma, 2025)](/src/context-rot.png)

<!--
Speaker notes:
The Chroma study tested GPT-4.1, Claude 4, Gemini 2.5, Qwen3 and 14 others.
The finding: every model's performance degrades as input length grows.
Not some. Not most. All of them.
And critically: the degradation starts well below the stated context limit.
A model with a 200k token window may show meaningful degradation at 50k tokens.
The decline is a gradient, not a cliff. Which makes it insidious:
the model still produces output. It just produces worse output.
You don't get an error. You get a subtly wrong answer.
Context rot is what happens when stale, contradictory, or irrelevant content
accumulates over time. The context doesn't just get full. It gets noisy.
Worth knowing for Q&A: the original report was July 2025, and the finding has
since been reproduced outside question answering — long-horizon search agents
and classifier monitors show the same curve. It's a property of long inputs,
not a quirk of one benchmark.
-->

---

# Context rot — the cost of irrelevant content

Same question. Same model. Same answer exists in the context.

The only difference: one prompt is <span class="accent">300 tokens</span>. The other is <span class="accent">113,000</span>.

![LongMemEval — focused vs full context performance gap (Chroma, 2025)](/src/context-rot-longmemeval.png)

<!--
Speaker notes:
This is the LongMemEval experiment. Models were given the same questions twice.
Once with a focused prompt of ~300 tokens: only the relevant conversation.
Once with the full prompt of ~113k tokens: the entire chat history.
The bars show the performance gap.
The model knew the answer. It just couldn't find it reliably in the noise.
For Claude specifically the gap is dramatic: Sonnet 4 and Opus 4 are so
conservative under ambiguity that they often abstain entirely rather than guess.
They score lower not because they hallucinate but because they refuse to commit
when they can't locate the answer cleanly in 113k tokens of context.
This is context rot made concrete: not a theoretical degradation curve,
but a real performance gap on real questions with real answers present.
Left unmanaged, context doesn't just fill up. It rots.
This is exactly what the techniques in the next section are designed to prevent.
-->

---

# Types of memory

| Type               | What it is                                     | Lives where               | Who writes it            |
| ------------------ | ---------------------------------------------- | ------------------------- | ------------------------ |
| Working memory     | Current session state, active task             | Context window            | The model, automatically |
| Episodic memory    | Event logs, session history                    | Experiences / logs        | The runtime, by default  |
| Semantic memory    | Declarative knowledge, architecture, decisions | Reference documents       | You, deliberately        |
| Procedural memory  | Skills, rules, behavioral instructions         | System prompt, rule files | You, deliberately        |
| Associative memory | Patterns, preferences, learned behaviors       | Preferences / profiles    | Accrues from use         |

The column that decides everything is the last one. <span class="accent">The layers worth investing in are the ones nobody writes unless you do.</span>

<!--
Speaker notes:
Not all memory is the same problem.
Working memory is what the agent sees right now. It's in the context window.
It's fast, it's precise, and it resets completely when the session ends.
Episodic memory is what happened. Session logs, conversation history.
Useful for recovery and audit. Too expensive to load by default.
Semantic memory is what the agent knows. Architecture decisions, conventions,
project structure. This is the layer most worth investing in.
It's also the layer most developers neglect.
Procedural memory is how the agent behaves. System prompts, rule files, CLAUDE.md.
This is where behavioral consistency lives across sessions.
Associative memory is what the agent has learned about you.
Preferences, patterns, working style. Cross-session, cross-project.
The techniques we'll cover apply across all five types,
but the highest leverage is on semantic memory — it's the layer
that compounds most clearly as a project grows.
Four of these five persist beyond the session. Only working memory doesn't.
That split is the next slide.
-->

---

# Short-term vs long-term memory

| | Short-term (working) | Long-term (persisted) |
| ---------- | ----------------------------- | ------------------------------------- |
| Scope | One session | Across sessions, tools, projects |
| Lives in | The context window | Files, database, memory service |
| Cost | Tokens on every single turn | Storage, plus a load when you ask |
| Fails by | Rot, distraction, overflow | Staleness, drift, contradiction |
| Cleared by | Closing the session | Only by you |

<div class="callout">

The context window is not memory. It's the desk. <span class="accent">Long-term memory is the filing cabinet.</span>

</div>

<!--
Speaker notes:
This is the distinction the whole talk rests on, so make it explicit.
Everything we've discussed so far — rot, lost in the middle, the seven layers —
is a short-term memory problem. It's about what's on the desk right now.
Long-term memory is a different problem with different failure modes.
Short-term memory fails loudly: it fills up, it rots, the session ends.
Long-term memory fails quietly: it goes stale, it drifts from reality,
it starts contradicting the code it describes. Nobody notices until
the agent confidently builds on a decision you reversed two months ago.
The important beat: you cannot fix a long-term memory problem
by buying a bigger context window. A bigger desk does not file anything.
Stateless by default is the short-term reality.
Stateful by design is the long-term choice.
-->

---

# The long-term memory lifecycle

Four operations. Most setups implement two.

1. **<span class="accent">Write</span>** — what earns a place? Decisions, conventions, constraints. Not transcripts.
2. **<span class="accent">Read</span>** — retrieved by index, for a named phase. Not everything, not by default.
3. **<span class="accent">Update</span>** — two speeds. State is overwritten; decisions are marked `superseded`, never deleted. Drop the *why* and the rejected option comes back in three months.
4. **<span class="accent">Forget</span>** — expiry and demotion are features. Memory that only grows eventually stops being loadable.

<div class="callout">

Write and read make a memory system look like it works. <span class="accent">Update and forget are what make it trustworthy.</span>

</div>

<!--
Speaker notes:
Four operations. Almost everyone builds the first two and stops.
Write: the hard question isn't how, it's what. The filter is durability.
Would this still be true in three months? A decision is durable.
A conversation is not. If you write everything, you've built a log,
and we already said logs are expensive for agents.
Read: covered by index-first and phase-based loading. Named file, named phase.
Update: this is the one people skip, and it's the one that causes damage.
You changed payment provider. The old note still says Stripe.
Now the agent has two answers and no way to rank them.
That's context clash — except it arrives weeks later, from your own memory layer.
The rule: memory records are overwritten, not versioned. Git has your history.
Forget: the counterintuitive one. Deleting memory is not losing knowledge.
Anything you might need again lives on demand, in logs, in git.
What lives in the default load has to earn that seat every session.
If nothing ever leaves, the load cost only goes up, and one day
the memory layer costs more to read than it saves.
-->

---
layout: section
---

# The Techniques

<!--
Speaker notes:
Five techniques. Each one solves a specific failure mode.
The first four are about what goes into the context. The fifth is about what stays out.
The scenario: a developer resuming work on a feature after closing the session yesterday.
Simple, universal. No special tooling required.
We'll see what changes at each step.
-->

---

# 1. Index-first loading

<div class="callout">

**Prevents:** cold start confusion — the agent doesn't know where it is or what it's doing.

</div>

**The problem:** You reopen your session. The agent has no orientation.
What project is this? What was the last decision? Where did you leave off?

**The technique:** Before reading anything else, load one small file
that answers those questions. A dispatch table, not a document.

```md
# AGENTS.md — payments-refactor

- Current state → `docs/state.md` (read first)
- Architecture → `docs/architecture.md`
- Specs → `docs/specs/` (one file per feature)
- Payment providers → `docs/notes/stripe-integration.md`
- Conventions → `docs/conventions.md`
```

This has a name now: **`AGENTS.md`** — stewarded by the Agentic AI Foundation under the Linux Foundation since December 2025, read natively by 25+ coding agents, used by <span class="accent">60,000+ open-source projects</span>.

<!--
Speaker notes:
The key insight here is the word "dispatch table."
This file does not contain the knowledge. It points to where the knowledge is.
INDEX.md in a filesystem, a project manifest, a README-style map — the format doesn't matter.
What matters is that it's small, always current, and always the first thing loaded.
The cost of orientation drops from "re-read everything" to "read one file."
The developer resumes instantly. No re-explanation needed.

CLAUDE.md and AGENTS.md are a special case of this file: the main agent reads them
automatically at session start. No instruction needed, no tool call. They're already in context.
That's exactly why they should stay an index — small, pointing elsewhere — not a knowledge dump.

Bonus connection: this technique also maximizes prompt cache hits.
Stable content loaded first means the cache prefix stays consistent across sessions.
Index-first loading is also cache-first loading. Same discipline, two benefits.
-->

---

# 2. Anchored iterative summarization

<div class="callout">

**Prevents:** context rot — accumulated noise degrading what the agent sees over time.

</div>

**The problem:** Yesterday's session produced a long conversation transcript.
Transcripts are noisy, long, and expensive to load.

**The technique:** Summarize into a structured state document.
<span class="accent">Overwrite it every session. Never append.</span>

```md
## Goal

Redesign checkout flow to reduce drop-off at payment step.

## Explored

- Option A: single-page checkout (rejected: too much refactor)
- Option B: progressive disclosure (selected)
- Option C: modal overlay (rejected: mobile UX concerns)

## Open

- Payment provider: Stripe vs Braintree undecided
```

<!--
Speaker notes:
"Anchored" means the document has a fixed structure that never changes.
Goal, done, next, blocked, open questions. Always in the same place.
"Iterative" means it gets rewritten at the end of every session, not appended.
This is the key discipline: overwrite, never append.
If you append, the document grows. It becomes a log.
Logs are useful for humans. They are expensive for agents.
An overwritten state document stays bounded, stays useful, stays cheap to load.
The developer's next session loads this instead of a 50-turn transcript.
Structured, dense, loadable in seconds.
-->

---

# 3. Phase-based context loading

<div class="callout">

**Prevents:** context distraction — the agent over-relies on irrelevant history instead of reasoning clearly.

</div>

**The problem:** The agent doesn't need everything at once.
Loading everything upfront wastes context window space.

**The technique:** Load in phases. Each phase loads only
what the current step requires.

| Phase          | What loads                          |
| -------------- | ----------------------------------- |
| Orientation    | Index file + state document         |
| Task           | Relevant specs + architecture notes |
| Implementation | Specific files being modified       |

<!--
Speaker notes:
Context window is RAM. It's limited and it resets.
The instinct is to load everything upfront so the agent is never missing information.
The problem is that context loaded but never used still occupies space.
And space occupied early is space unavailable later when you actually need it.
Phase-based loading flips this: start minimal, load more as the task demands it.
The developer's session starts with orientation only.
Once the agent knows what it's doing, it loads the relevant specs.
Only when writing code does it load the specific files being modified.
Three phases. Three cost levels. No waste.
-->

---

# 4. Just-in-time file retrieval

<div class="callout">

**Prevents:** context bloat — paying the token cost of files the current task never needs.

</div>

**The problem:** When implementing one module, the agent doesn't need
the entire project history or every architecture document.

**The technique:** Load files at the moment they're needed,
not before. Everything else stays on disk.

```
Implementing: checkout/PaymentStep.tsx
Loads: architecture.md, PaymentStep spec, stripe-integration notes
Does not load: full session history, unrelated modules, past decisions
```

<!--
Speaker notes:
This is the most tactical of the four techniques.
It follows directly from phase-based loading but applies at the file level.
The agent knows what it's working on. It loads what that work requires.
Not the project. Not the session history. The specific task.
Combined with the previous three techniques, that's the load path solved:
the agent loads exactly what it needs, when it needs it,
at a cost that stays predictable as the project grows.
But there's a gap left, and it's the one the next technique closes.
All four of these assume you know what to load. Finding that out
is itself expensive, and everything you read while searching stays behind.

Q&A preparation — four named failure modes worth knowing cold:
- Context poisoning: a hallucination or bad tool output enters the context
  and gets compounded over subsequent steps. Fixed by pruning and validation.
- Context distraction: context grows so long the model over-relies on recent
  history and stops reasoning from first principles. Fixed by compression.
- Context confusion: too many tools or irrelevant content causes the model
  to call the wrong tools or produce low-quality outputs. Fixed by selection.
- Context clash: new information contradicts something already in context,
  producing inconsistent behavior. Fixed by establishing a clear authority order:
  system prompt > retrieved facts > conversation history.
-->

---

# 5. Sub-agent isolation

<div class="callout">

**Prevents:** context pollution — the search for the answer costing more than the answer.

</div>

**The problem:** To find one file, the agent reads twenty.
Nineteen of them stay in the window for the rest of the session.

**The technique:** Run the dirty work — exploration, log triage, wide search —
in a separate context. Only the conclusion comes back.

```
Sub-agent reads:      20 files, ~40k tokens
Returns:              "Session handling: src/auth/session.ts:40, JWT + refresh"
Main context pays:    one line
```

Write · Select · Compress · <span class="accent">Isolate</span> — the four canonical context operations.
Techniques 1&ndash;4 cover the first three.

<!--
Speaker notes:
This is the technique people discover last and miss most.
The other four are about what you put in. This one is about what you keep out.
Start with the problem, because everyone has lived it.
You ask where session handling lives. The agent greps, opens a file,
wrong one, opens another, reads a config, follows an import.
Twenty files later it tells you the answer — and nineteen of those files
are now sitting in your context window, permanently, for a question
that was answered in one line.
That's not a retrieval failure. The retrieval worked. The cost is the residue.
The fix is to give the search its own window. The sub-agent burns forty
thousand tokens finding the answer, and the only thing that crosses
back into your session is the sentence you actually asked for.
The trade is real and worth naming: the sub-agent doesn't have your context,
so it can't use what you already established. You pay for isolation
in re-explanation. For wide, dirty, throwaway work, that trade is almost always good.
For work that needs the thread of the conversation, it isn't.
Then the closing line. The field has settled on four operations:
write, select, compress, isolate. Techniques one through four
were write, select and compress. This is the fourth.
If someone asks what to adopt first — it's this one, because it's the only
technique that gets cheaper as the project gets bigger.
-->

---
layout: section
---

# The Decision Space

<!--
Speaker notes:
The techniques are the what.
Now the decisions every practitioner faces when implementing them.
These are not specific to any one tool or workflow.
They're the decisions you'll face regardless of how you build this.
-->

---

# Where does memory live?

Three options. Each with a different tradeoff.

| Location        | Benefit                           | Cost                                                |
| --------------- | --------------------------------- | --------------------------------------------------- |
| Inside the tool | Zero setup                        | Locked to one tool                                  |
| Inside the repo | Version controlled                | Pollutes git history, couples knowledge to codebase |
| Outside both    | Tool-agnostic, survives archiving | Requires discipline to maintain                     |

<!--
Speaker notes:
This is the first decision and it shapes everything else.
Inside the tool: easiest to start, hardest to escape.
Your memory is now owned by the vendor. Switch tools, lose memory.
Cursor Memories, for example, are deliberately per-project.
When you switch to Claude Code or a local model, they don't come with you.
Inside the repo: feels natural, version controlled, reviewable.
But your architectural decisions end up in git alongside your source code.
And when you archive the repo, the knowledge goes with it.
Outside both: the hardest to set up, the most resilient long-term.
Your knowledge layer is independent of any tool, any repo, any vendor.
It lives as long as you maintain it.
There's no universally right answer. But the tradeoffs are clear.
Know which one you're choosing and why.
-->

---

# What storage primitive?

The instinct is to reach for infrastructure.

- Vector store — semantic search, embeddings, retrieval pipelines
- Database — structured queries, relationships, versioning
- MCP server — tool integration, manifest-based discovery

<div class="callout">

**The question to ask first:** _Do I have a search problem, or a retrieval problem?_

</div>

<!--
Speaker notes:
This is where most people over-engineer.
Vector stores are powerful. They're also complex, expensive, and optimized for search.
But context engineering, at least for the use cases we've been discussing,
is not usually a search problem.
You almost always know exactly which file you need.
The state document for this project. The architecture notes for this module.
If you know where the file is, you don't need search. You need retrieval.
And retrieval doesn't require infrastructure.
A filesystem with discipline is often enough.

The MCP point is worth a separate beat:
MCP servers solve a real problem — tool integration, discovery.
But every MCP call loads a tool manifest into your context window.
Some manifests are 7 tools. Some are 1,200.
At 1,200 tools, every single message in a session costs extra tokens
just for the manifest. That's a tax on every interaction, paid whether
you use those tools or not.
Know what you're paying before you commit to the infrastructure.

Ordering rule (connects to prompt caching):
Stable content always goes first: index file, architecture notes,
anything that doesn't change between turns. Dynamic content goes last.
Providers cache the stable prefix and reuse it across requests.
Index-first loading is also cache-first loading.
-->

---

# How do you keep memory from becoming a liability?

Memory that grows without constraint becomes:

- Too expensive to load
- Too noisy to be useful
- Too stale to be trusted

**Three disciplines:**

1. Overwrite state, never append
2. Cap reference documents at a fixed line count
3. Load experiences on demand, never by default

<!--
Speaker notes:
This is the maintenance problem. Every memory system faces it eventually.
The techniques we covered are partly about this: anchored iterative summarization
is specifically designed to prevent unbounded growth.
But the principle applies beyond state documents.
Any file that agents load regularly needs a size contract.
If it can grow forever, it will. And eventually it will cost more than it's worth.
The third point is subtle but important: session logs, experience entries,
historical decisions — these are valuable. But they should never load by default.
They load when you ask for them. The default load stays small and fast.
-->

---

# How do you know your context is working?

You'll notice the symptoms before you notice any metric:

- The agent re-asks something you already answered
- It contradicts a decision from the same session
- It reaches for the wrong tool
- Answers get vaguer the longer the session runs

**Three numbers worth watching:**

- <span class="accent">Orientation cost</span> — tokens loaded before any work starts
- <span class="accent">Utilization</span> — how much of the loaded context the task actually touched
- <span class="accent">Useful turns</span> — how many turns before quality visibly drops

<!--
Speaker notes:
People ask "is my context engineering working?" and expect a dashboard.
Start with the honest answer: you'll feel it before you measure it.
Those four symptoms map one-to-one onto the failure modes from earlier.
Re-asking is a memory problem. Contradiction is clash.
Wrong tool is confusion. Vagueness over time is rot and distraction.
If you're seeing these, the diagnosis is already in the room.
Then the three numbers. None of them need infrastructure.
Orientation cost: what does it cost to start? If your first turn is 40k tokens,
you've spent a fifth of a 200k window before the agent has done anything.
Utilization: of everything you loaded, how much did the task actually use?
Low utilization means you're paying rent on context you didn't need.
Useful turns: at what point does quality drop? That number is your session budget.
When you know it, you stop being surprised by it — you compact before you hit it.
-->

---

# Test your context like you test your code

<div class="callout">

**The golden set:** ten to twenty questions that <span class="accent">only your memory layer can answer.</span>

</div>

```md
Which payment provider did we choose, and what did we reject?
What is the naming convention for API route handlers?
Where did the last session stop, and why?
```

Ask them at the start of a session, from cold.
If the agent can't answer from what it loaded, the memory layer failed — not the model.

- **Block on regression, not on score** — compare against the last known-good run
- **Ablate** — remove a file, halve the tools: does anything actually get worse?
- **Automate** — promptfoo or DeepEval can score the same set in CI on every change to your rule files

<!--
Speaker notes:
This is the part almost nobody does, and it's the cheapest thing in the talk.
The golden set is just questions with known answers that live in your memory layer.
Write ten. Twenty is plenty. Keep them in a file next to the memory itself.
The test is: open a cold session, load what you normally load, ask the questions.
The framing matters — if the answer isn't there, that's not a model failure.
You never gave it the information. That's a context bug, and context bugs are fixable.
Block on regression, not absolute score: LLM outputs are non-deterministic,
so an absolute pass threshold gives you flaky tests and lost trust.
Score today against the last good run and fail on the delta instead.
Ablation is the underrated one, and it's how you find waste.
Delete a file from the default load and re-run. Nothing got worse?
It was never earning its seat. Same with tools — remember 46 versus 19.
Ablation is how you discover which half of your context was decoration.
And if your rules live in files — CLAUDE.md, rule files, an index —
those files are code. They change behavior. They deserve the same gate
as anything else you'd never merge untested.
-->

---

# How the field measures memory

| Benchmark       | What it tests                                           | Scale                        |
| --------------- | ------------------------------------------------------- | ---------------------------- |
| LoCoMo          | Multi-session conversational recall                     | 50 conversations, ~300 turns |
| LongMemEval     | Knowledge updates, temporal reasoning, multi-session     | 500 questions, 6 categories  |
| BEAM            | Memory behavior at production volume                     | 1M and 10M tokens            |

The benchmark everyone quotes — needle in a haystack — is the one that <span class="accent">most overstates</span> how healthy long context is.

<!--
Speaker notes:
Brief slide. This is for the people who want to go read something after.
Three benchmarks define the memory conversation right now.
LoCoMo is the most widely reported number across memory systems:
fifty long multi-session conversations, roughly three hundred turns each.
LongMemEval is the more demanding one — five hundred questions,
and critically it tests knowledge updates: what happens when a fact changes.
That's the update operation from two slides ago, measured.
BEAM is the production-scale one, at one and ten million tokens.
The point of BEAM is that you cannot solve it by buying a bigger window.
And the honesty note at the bottom: needle in a haystack is the benchmark
everyone cites and models score well on it, which is exactly why
long context looks more solved than it is. NIAH measures lexical retrieval.
Real tasks need reasoning across what was retrieved. Those are not the same test.
If someone in Q&A cites a big NIAH number, that's the answer.
-->

---
layout: section
---

# What this enables

---

# For a single developer

The problem we started with: you reopen a session and the agent starts cold.

**With context engineering:**

- Resume a feature after three days with no re-explanation
- The agent knows the last decision, the open questions, the next step
- Switching models costs nothing: the context travels with you, not with the tool

<!--
Speaker notes:
This is the minimum viable version of what we've been building toward.
One developer. One project. One discipline.
The techniques don't require infrastructure. They require habit.
Write the state document at the end of every session.
Load the index file at the start of every session.
That's it. Everything else follows.
-->

---

# At larger scale

The same discipline, applied further.

- A team sharing context across sessions and members
- Multiple tools reading the same memory layer: Claude Code, Cursor, a local model
- A shared write layer — an MCP server all tools call to write the same vault

<!--
The last point is not optional for cross-tool scenarios.
Tools don't share files automatically.
If you want Claude Code and Cursor and a local model to write
to the same memory layer, you need a shared interface.
MCP is that interface. Not a nice-to-have. The required piece.
-->

<!--
Speaker notes:
This is where the three failure modes from the opening become team problems, not just personal ones.
Session amnesia: the new team member starts cold on a project that's been running for months.
Tool switch loss: the developer using Cursor doesn't have access to what the Claude Code user built.
Cross-project blindness: the architectural decision made on project A never reaches project B.
The context engineering discipline is the same at every scale.
What changes is the write layer.

On the MCP point:
MCP as a write layer is now standard practice, not experimental.
Mem0, agentmemory, OpenMemory — these projects all use MCP as the interface
through which different tools write to a shared memory layer.
The tools don't talk to each other. They talk to the same memory layer.
That's the architecture. The discipline we covered is what makes the memory layer useful.
-->

---

# Four approaches, by what they treat memory as

| Approach                  | Memory is...                          | Example                  | Lock-in            |
| ------------------------- | ------------------------------------- | ------------------------ | ------------------ |
| Managed memory API        | Extracted facts, indexed for you      | Mem0, Supermemory        | The vendor's store |
| Temporal knowledge graph  | Facts with a *valid from* / *until*   | Zep / Graphiti           | Graph schema + DB  |
| Memory-first agent runtime| The agent's own editable state        | Letta (ex-MemGPT), Cognee| The whole runtime  |
| Filesystem-native         | A file                                | AGENTS.md + markdown     | None. It's a repo  |

Only one of these answers the staleness problem from two slides ago by design: <span class="accent">the temporal graph knows *when* a fact was true.</span>

<div class="callout">

_The discipline is the same. The write layer is a choice._

</div>

<!--
Speaker notes:
The useful way to read this table is not by how it's deployed —
cloud, self-hosted, local — because that's a vendor question.
Read it by what each one thinks memory actually is. That's the design question.
Row one, managed memory API: memory is facts, extracted from your conversations
and indexed for you. Mem0 is the most adopted of these. You get a drop-in
write and read API and you stop thinking about storage. The trade is
that the extraction policy is theirs, and the store is theirs.
Row two, temporal knowledge graph. This is the one worth slowing down on,
because it's the direct answer to the staleness problem from earlier.
Zep and Graphiti store facts as a graph with validity intervals —
not just "we use Stripe" but "we used Stripe, from March until September."
Every other approach on this slide overwrites the old fact or leaves it to rot.
This one knows when it was true, and that's a genuinely different capability.
The cost is a real graph, a real database, and an LLM pass on every write.
Row three, memory-first runtime: memory isn't a service you call,
it's the agent's own state, which the agent edits itself. Letta — formerly
MemGPT — is the reference implementation. The strongest version of the idea,
and also the biggest commitment: you're not adopting a memory layer,
you're adopting the runtime that runs your agent.
Row four: memory is a file. No extraction, no schema, no service.
And this is the honesty point — fourteen months ago this was the weakest row
on the slide, the one for people who didn't want infrastructure.
It isn't anymore. AGENTS.md made plain markdown the most portable option here,
not the least, because every tool can read it and nothing owns it.
Then the lock-in column, which is the whole reason the slide exists.
Read it top to bottom: the vendor's store, a graph schema, an entire runtime, nothing.
That's not an argument for the bottom row. It's an argument for knowing
which one you signed up for. All four work. None of them work
without the context engineering discipline underneath.
The infrastructure is a choice. The discipline is not optional.
-->

---

# Fourteen months

| | Mid-2025 | Now |
| ------------------ | ------------------------------------- | --------------------------------------------------- |
| The long-context fix | "Wait for a bigger window" | 1M windows shipped — they still sag before 100k |
| Tool definitions | Full manifest, every turn, forever | Deferred loading: one case went 150k → 2k tokens |
| Trimming context | You did it by hand, between sessions | The agent edits its own context, mid-session |
| Agent memory | Markdown files and personal discipline | Managed memory layers with per-write audit logs |
| The scary attack | Prompt injection — gone when you close the tab | Memory poisoning — it waits for you in tomorrow's session |

<!--
Speaker notes:
This slide exists to make one point: the ground moved under this talk while I was writing it.
Fourteen months. Roughly the distance between the Chroma context rot report
and this room.
Go row by row, fast, don't linger.
Row one: the honest answer in 2025 was "the window will get bigger."
It did. A million tokens. And measurement still shows the sag starting
well before a hundred thousand. Bigger windows moved the middle. They didn't remove it.
Row two: the tool manifest tax I mentioned earlier was just the cost of doing business.
Now tool definitions can load on demand. The reported numbers are not incremental —
one published case went from a hundred and fifty thousand tokens to two thousand.
Row three: compaction used to be homework you did between sessions.
Now the agent can clear its own stale tool results while it works.
Row four: memory was a folder of markdown files and the discipline to maintain it.
Now there are managed memory layers, with audit logs on every write.
Row five is the one to slow down on. In 2025 the attack you worried about
was prompt injection, and it died when the session ended.
Now the attack writes itself into your memory layer and waits.
That's the new shape of the problem: persistence cuts both ways.
Then the turn: so if everything on this slide changed in fourteen months,
what was worth learning?
-->

---

# What didn't change

Every row on that slide was an <span class="accent">implementation</span> moving.

The questions underneath stayed exactly where they were:

- What does the agent need to know right now?
- What has to survive the session ending?
- What earns its seat in the default load?
- Who is allowed to write to what the agent believes?

<div class="callout">

The tools got a year better. <span class="accent">The decisions are still yours.</span>

</div>

<!--
Speaker notes:
This is the payoff for the previous slide, and it's the thesis of the talk.
Every single thing that changed was an implementation detail.
Better windows, cheaper tools, self-editing context, managed memory.
Not one of them answered a question on this list.
A million-token window doesn't tell you what belongs in it.
A managed memory service doesn't tell you what's worth remembering.
Audit logs don't tell you who should be trusted to write.
These four questions were the job in 2025 and they're the job now.
That's why I'd bet on learning the discipline over learning any tool:
the tool you learn today has a half-life measured in months.
The questions don't.
If someone asks "won't this all be automated soon" — this is the answer.
The automation keeps arriving. It keeps not deciding what your agent should know.
-->

---

# Close

The tools are already there. The models are already capable enough.

Context engineering is not something you install.

**<span class="accent">It's how you think about what your agents know.</span>**

<!--
Speaker notes:
Land this slowly.
The audience has the techniques. They have the decision space.
The thing I want them to leave with is simpler than any of that:
this is not a tool problem. It's a thinking problem.
Once you have the mental model, the implementation follows.
Whatever tools they use, whatever models, whatever workflow:
the discipline is the same.
Decide what your agents know. Decide when they know it.
Make sure that knowledge survives the boundary.
That's context engineering.
Thank the room. Open for questions.
-->

---
layout: center
---

<div class="flex flex-row items-center gap-12 w-full">

  <div class="flex flex-col gap-5 flex-1">

  <div class="thank-you-title">Thank you</div>

**Giorgio Galassi** <br>
Senior Frontend Engineer (Freelancer) · GDG Roma Città Organizer

  <div class="tags">
    <span class="tag">Angular</span>
    <span class="tag">AI Workflows</span>
    <span class="tag">Context Engineering</span>
  </div>

  <div class="flex flex-col items-start gap-2">
    <img src="/src/qrcode.png" alt="QR code" class="qr-code w-48 h-48" />
    <span class="text-sm text-gray-400">Links, slides & more</span>
  </div>

  </div>

  <div class="flex-shrink-0">
    <img src="/src/profile.png" alt="Giorgio Galassi" class="rounded-full w-60 h-60 object-cover" />
  </div>

</div>

<!--
Speaker notes:
Leave contact or social handle here if desired.
Open Q&A.
-->
