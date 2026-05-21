# Multi-Agent Orchestration: Patterns, Practice, and Snowflake

> A field guide to the multi-agent design space — the canonical pattern catalog, how those patterns actually compose inside Snowflake today, and the one place where you still need to reach outside the platform.

---

## Table of Contents

**Part I — The Pattern Catalog**
- [Why Orchestration Patterns Matter](#why-orchestration-patterns-matter)
- [The Canonical Five (Anthropic)](#the-canonical-five-anthropic)
- [Cross-Framework Alias Table](#cross-framework-alias-table)
- [Additional Patterns From Other Frameworks](#additional-patterns-from-other-frameworks)
- [Industry Examples By Topology](#industry-examples-by-topology)
- [Collaboration Mode (Orthogonal Axis)](#collaboration-mode-orthogonal-axis)
- [Choosing a Pattern — Quick Heuristics](#choosing-a-pattern--quick-heuristics)
- [Common Tradeoffs](#common-tradeoffs)

**Part II — Multi-Agent Orchestration in Snowflake, Today**
- [Pattern 1: One Cortex Agent, Many Tools](#pattern-1-one-cortex-agent-many-tools)
- [Pattern 2: When Two Agents Are Genuinely Required](#pattern-2-when-two-agents-are-genuinely-required)
- [Pattern 3: When the Workflow Is the Orchestrator](#pattern-3-when-the-workflow-is-the-orchestrator)
- [Pattern 4: When a Real Framework Is Required](#pattern-4-when-a-real-framework-is-required)
- [Pattern 5: When the Human Is Part of the Orchestration](#pattern-5-when-the-human-is-part-of-the-orchestration)
- [Pattern 6: When Scale Is Being Mistaken for Complexity](#pattern-6-when-scale-is-being-mistaken-for-complexity)
- [Composition: 30-Day Readmission Prevention](#composition-30-day-readmission-prevention)
- [A Note on Memory](#a-note-on-memory)
- [So Which Pattern Fits?](#so-which-pattern-fits)

**Part III — The A2A Wrapper: When Snowflake Has To Talk To The Outside Mesh**
- [High-Level Architecture](#high-level-architecture)
- [Why This Matters](#why-this-matters)
- [HCLS Worked Example: Prior Authorization](#hcls-worked-example-prior-authorization)
- [Why We Cannot Do This Inside Snowflake Today](#why-we-cannot-do-this-inside-snowflake-today)
- [The Forward Look](#the-forward-look)

---

# Part I — The Pattern Catalog

This part of the post is an attempt to surface patterns *beyond* orchestrator-workers. That shape — a central planner decomposing work and farming it out — is almost always the first thing that comes to mind when someone says "multi-agent system," and it's often the wrong default. The sections below list roughly fifteen other named patterns in active use across the major frameworks, several of which are simpler, cheaper, or more reliable for the workloads they fit. Many of those names are aliases for the same underlying topology — the [cross-framework table](#cross-framework-alias-table) further down collapses them into six core shapes — but the names themselves are worth knowing because each ecosystem ships its own.

It's also an attempt to flatten the vocabulary. Multi-agent systems have exploded as a design space over the last two years, and with the explosion has come a Tower-of-Babel problem: every framework names its own version of the same handful of ideas, and the few patterns that *are* genuinely distinct get lost in the noise. The sections below survey the patterns the major players have published, line up the aliases, and end with a [practical guide](#choosing-a-pattern--quick-heuristics) to which one to reach for.

## Why Orchestration Patterns Matter

Knowing the full set of multi-agent orchestration best practices is not a stylistic concern — it directly determines unit economics, latency, reliability, and what you can actually ship to production. The reasons to care are stackable, and most teams underweight all of them:

1. **Cost–performance frontier.** The biggest lever in any multi-agent system is *which model runs which step*. The goal is to push as much work as possible onto the cheapest tier of model that is still sufficient for that subtask, and reserve frontier-tier calls for the steps that actually need them. Pattern choice is what makes that possible. A *Routing* pattern can send 80% of traffic to a small model and only escalate ambiguous cases to a frontier model — often cutting cost by an order of magnitude with no measurable quality loss. A *Prompt Chaining* pipeline lets you mix tiers per step (e.g., Haiku-class for extraction, Sonnet-class for reasoning, Haiku-class again for formatting). An *Evaluator–Optimizer* loop can use a small generator with a slightly larger critic and beat a single large model on both cost and quality. Pushed to a single large autonomous agent instead, the same workload pays frontier-tier prices on every token, including the trivial ones.
2. **Latency budgets.** *Parallelization* turns a sequence of independent calls into a single wall-clock step. For user-facing flows where p95 latency is a hard constraint (chat, voice, real-time triage), this is often the only way to hit budget without dropping work. Conversely, autonomous agents with unbounded tool-call loops have effectively unbounded latency — fine for batch, fatal for interactive.
3. **Reliability and auditability.** Regulated workloads (claims adjudication, prior-auth, quality inspection) need every step to be reconstructable after the fact. *Fixed-ordered* and *Central-decomposer* shapes give you a deterministic trace; *Swarm* and *Network* shapes do not. Choosing the wrong pattern here is not a performance problem, it's a compliance problem.
4. **Failure-mode containment.** Each pattern has a characteristic failure mode: chains cascade errors, swarms lose context on handoff, supervisors degrade as the agent count grows past ~8–10, evaluator loops can oscillate. Knowing the catalog means you can pick the failure mode you can tolerate, instead of inheriting whichever one the default pattern happens to have.
5. **Debuggability and iteration speed.** Simpler topologies are dramatically faster to debug. A team that ships a *Prompt Chain* can usually localize a regression to a single step in minutes; a team that ships an autonomous agent often cannot localize it at all without replaying full traces. This compounds across every iteration cycle.
6. **Specialist composition.** Some problems are genuinely cross-functional (virtual tumor board, engineering change review) and the value comes from having distinct expert prompts/tools/contexts disagree productively. *Shared-thread* and *Debate* shapes are how you get that. A single agent can't simulate disagreement against itself reliably.
7. **Bounded autonomy.** The right pattern lets you give individual agents *more* autonomy without paying for it in risk or cost. A free-running autonomous agent with broad tool access has to be conservative everywhere, because any single bad tool call can corrupt state or burn budget. Wrap that same agent inside a *Central-decomposer* with scoped tool subsets per worker, an *Evaluator–Optimizer* gate, or a *Hierarchical* tree where each leaf only sees its slice of the world, and you can safely turn the autonomy dial up at the leaves — they explore aggressively inside a small blast radius while the surrounding pattern enforces global invariants. This is how you get the upside of agentic behavior (self-correction, dynamic tool use, recovery from partial failure) without the usual downside (runaway loops, unbounded spend, irreversible side effects).
8. **When a single agent with tools genuinely is not enough.** The default move — start with one agent and a well-curated tool belt — is right roughly 80% of the time. The remaining 20% is real, and the failure modes are recognizable. The signal that you have crossed into genuinely-multi-agent territory is usually one of these:
   - **Tool-belt overload.** Past roughly 20–30 tools in one agent, planning quality collapses. The model spends more tokens deciding *which* tool than *running* one, picks the wrong tool more often, and confuses similar-named tools. The fix is a *Hierarchical tree* or *Routing* layer that narrows the tool set the leaf sees.
   - **Conflicting reasoning styles in one context.** Clinical reasoning, codegen, and policy citation each want different system prompts, different temperature, sometimes different models. Pushed into a single agent they degrade each other; split into specialists they each get sharper.
   - **Governance and blast-radius asymmetry.** One subagent reads PHI, another writes to the EHR, a third calls a partner API. RBAC, audit, and rate-limiting want them as separate identities. Merging them into one agent collapses controls the organization specifically created.
   - **Different ownership and release cadence.** The medical-policy team ships weekly; the ops team that authors letters ships nightly. One agent forces one cadence. Two agents lets each team move at its own speed and roll back independently.
   - **Productive disagreement.** Some answers (a tumor-board recommendation, a high-stakes engineering change, a contested policy interpretation) are *better* when two specialists with different priors argue and a judge decides. A single agent can't simulate disagreement against itself reliably; *Debate* and *Shared-thread* shapes are how you get it.
   - **Long-horizon memory pressure.** When the working set genuinely exceeds what a single context can carry — months of patient history, years of policy precedent — splitting the work across specialists with their own scoped memory beats stuffing everything into one prompt.

   Sketch — the failure mode this point is trying to name:

   ```
     SINGLE AGENT (works up to a point)             MULTI-AGENT (when it does not)

       ┌────────────────────┐                          ┌──────────────┐
       │  one agent         │                          │  router /    │
       │  one prompt        │                          │  supervisor  │
       │  one context       │                          └──┬───┬───┬───┘
       │  N tools ──────────┼──▶ planning             ┌───┘   │   └───┐
       │            (good)  │    quality              ▼       ▼       ▼
       └────────────────────┘    cliff @ ~20–30   specialist  specialist  specialist
                                 tools               (clinical) (codegen)  (letters)
                                                       │           │          │
                                                  scoped tools  scoped tools  scoped tools
                                                  scoped memory scoped memory scoped memory
                                                  scoped RBAC   scoped RBAC   scoped RBAC
   ```

   The bar to clear is: would a senior engineer reading the design *immediately* see why two agents are required, not just convenient? If yes, go multi-agent. If not, stay single-agent and add tools.

A concrete example of (1) in practice: a customer-support triage system built as *Routing* + *Prompt Chaining* — a small classifier model routes by intent, a small extractor pulls entities, a mid-tier model drafts the reply, and a small model formats it — will typically cost 5–20× less than the same workload handled by a single frontier-model autonomous agent, and will usually be faster and easier to evaluate per step. The same logic applies to document processing, code review, claims adjudication, and most batch analytics workloads. The pattern is the cost-control mechanism.

Sketch — the cost-tier idea, made visual:

```
   SINGLE FRONTIER AGENT                  TIERED PATTERN
   (one model does everything)            (model per step)

   ┌──────────────────────────┐           ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
   │  $$$$ frontier model     │           │ classify│─▶│ extract │─▶│ reason  │─▶│ format  │
   │  classify → extract →    │           │  $      │  │   $     │  │  $$$    │  │   $     │
   │  reason   → format       │           │ small   │  │ small   │  │ mid/big │  │ small   │
   │                          │           └─────────┘  └─────────┘  └─────────┘  └─────────┘
   │  every token at $$$$     │
   └──────────────────────────┘                  total spend ≈ 5–20× lower
```

The short version: pick the *simplest pattern that fits the task*, then pick the *smallest model that fits each step within that pattern*. The best practices below exist so you can do both.

## The Canonical Five (Anthropic)

**Workflows** are systems where LLMs and tools are wired together along a predefined code path. **Agents** are systems where the LLM dynamically directs its own steps and tool use. Most production "agent" systems are actually workflows in this sense, and that's usually a feature, not a bug. ([Anthropic — Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents))

Within the workflow side, Anthropic identifies five recurring shapes. They are worth memorizing because almost every other framework's pattern catalog can be expressed as a recombination of them.

- **Prompt Chaining** — fixed sequence of LLM calls, each consuming the previous output; optional gates/checks between steps. Best when a task cleanly decomposes into ordered subtasks. Trades latency for accuracy.
- **Routing** — classify input, dispatch to a specialized downstream prompt/tool. Best when input categories benefit from separate optimization.
- **Parallelization** — run LLM calls concurrently. Two variants: **Sectioning** (independent subtasks) and **Voting** (same task N times, aggregate).
- **Orchestrator–Workers** — central LLM dynamically decomposes, delegates to workers, synthesizes results. Use when subtasks can't be predicted upfront. This is the pattern most people have in mind when they say "multi-agent."
- **Evaluator–Optimizer** — generator LLM produces output; evaluator LLM critiques; loop until criteria met. Use when eval is clear and iteration measurably helps.

The six core topologies, side-by-side:

![Hand-drawn sketches of the six core multi-agent topologies: central decomposer, peer handoff, fixed ordered, concurrent, critique loop, shared thread](topology-sketches.svg)

Anthropic treats a sixth category separately because the autonomy profile is qualitatively different: the **Autonomous Agent**, an LLM that loops over tool calls and observations with no fixed DAG. It's the most flexible shape and also the most expensive and hardest to debug — which is why Anthropic's own advice is to start with the simplest workflow that works and only escalate when the task genuinely demands it.

## Cross-Framework Alias Table

Other frameworks ship their own pattern catalogs, but most of those names are aliases for the same handful of underlying topologies. The table below collapses everything down: columns are the major frameworks, rows are the underlying topology. Empty cells mean the framework either doesn't ship that shape as a first-class concept or expresses it through composition of others.

| Topology | Anthropic | LangGraph | AutoGen | OpenAI | Google ADK | CrewAI |
|---|---|---|---|---|---|---|
| Central decomposer | Orchestrator–Workers | Supervisor | MagenticOneGroupChat | Manager / Agents-as-Tools | Coordinator/Dispatcher | Hierarchical |
| Fixed ordered | Prompt Chaining | StateGraph chain | — | — | Sequential | Sequential |
| Peer handoff | — | Swarm | Swarm | Swarm | — | — |
| Concurrent | Parallelization | Branches/fan-out | — | Parallel tool calls | Parallel | — |
| Critique loop | Evaluator–Optimizer | Reflection | — | — | Generator–Critic / Iterative | — |
| Shared thread | — | — | RoundRobin/Selector GroupChat | — | — | — |

The two cells that surprise people are *Peer handoff* (Anthropic deliberately omits it; they don't recommend it as a default) and *Shared thread* (only AutoGen treats it as a built-in shape — everywhere else you have to assemble it from a graph and a message bus).

Sources: [LangGraph multi-agent](https://langchain-ai.github.io/langgraph/concepts/multi_agent/) · [AutoGen Teams](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/tutorial/teams.html) · [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/multi_agent/) · [Google ADK](https://google.github.io/adk-docs/agents/multi-agents/) · [CrewAI](https://docs.crewai.com/en/concepts/processes).

## Additional Patterns From Other Frameworks

![Hand-drawn sketches of additional multi-agent patterns: hierarchical tree, peer-to-peer network, manager with agents-as-tools, Magentic-One dual-loop, blackboard, and debate](additional-patterns-sketches.svg)

The five-plus-one above doesn't cover everything other frameworks have shipped. LangGraph, AutoGen, OpenAI's Agents SDK, Google's ADK, and CrewAI each surface variations that aren't in Anthropic's set — either entirely new shapes or refinements that meaningfully extend a canonical pattern. Pure renames (e.g. CrewAI's *Sequential* for Anthropic's *Prompt Chaining*) are folded into the alias table above and not repeated here.

- **Hierarchical (tree)** — *refinement of Orchestrator–Workers.* Once the agent count exceeds ~8–10, a single supervisor's routing prompts get unreliable; the fix is to nest supervisors so each only routes to a small set of subordinates. The new idea is the multi-level decomposition, not the supervisor itself. ([LangGraph Hierarchical Teams](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/hierarchical_agent_teams/))
- **Network / Peer-to-peer** — any agent can call any agent; no central coordinator. Flexible but hard to reason about. ([LangGraph multi-agent overview](https://langchain-ai.github.io/langgraph/concepts/multi_agent/))
- **Swarm (handoff)** — peer agents transfer control to each other via a "transfer to X" tool; routing is non-deterministic, decided by the active agent at runtime. The system just remembers who is "active." Stateless in OpenAI's original formulation. The crisp distinction from Network/Peer-to-peer: a network call is a function call (caller stays alive, control returns), while a swarm handoff is a baton pass (the previous agent exits and only one agent is active at a time). That's why network graphs can fan out into nested calls but swarm graphs are linear walks through the agent set — and why getting "back" to a known state in a swarm is hard once control wanders. ([OpenAI Swarm](https://github.com/openai/swarm) · [LangGraph Swarm](https://github.com/langchain-ai/langgraph-swarm-py) · [AutoGen Swarm](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/swarm.html))
- **Agents-as-Tools / Manager** — sub-agents exposed to a root agent as callable tools; root retains control rather than handing off. ([OpenAI Agents SDK – Multi-agent](https://openai.github.io/openai-agents-python/multi_agent/))
- **Group Chat (round-robin / selector)** — agents converse in a shared thread; next speaker chosen by rotation or by a selector LLM. ([AutoGen AgentChat Teams](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/tutorial/teams.html))
- **Magentic-One dual-loop** — orchestrator maintains an outer **Task Ledger** (facts, plan) and an inner **Progress Ledger** (per-step status); re-plans when stuck. ([Magentic-One](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html) · [paper](https://arxiv.org/abs/2411.04468))
- **Generator–Critic vs. Iterative Refinement** — *refinement of Evaluator–Optimizer.* Google ADK splits the loop in two: a *Generator–Critic* with a binary pass/fail gate, and *Iterative Refinement* with a qualitative-improvement objective. The split matters because the two need different stop conditions. ([Google ADK patterns](https://developers.googleblog.com/en/developers-guide-to-multi-agent-patterns-in-adk/) · [ADK docs](https://google.github.io/adk-docs/agents/multi-agents/))
- **Blackboard / shared memory** — agents read/write a shared state store instead of messaging each other directly. Classical AI pattern, re-surfacing in LLM systems. ([LangGraph state](https://langchain-ai.github.io/langgraph/concepts/low_level/))
- **Debate / Competitive** — multiple agents argue opposing positions; judge selects. Improves factuality on reasoning tasks. ([arXiv 2305.14325](https://arxiv.org/abs/2305.14325))

A few of these deserve emphasis. The **Manager / Agents-as-Tools** distinction matters because it preserves a single locus of control — the root agent never gives up the steering wheel — which is much easier to reason about than peer handoff. **Magentic-One**'s dual-ledger idea is the most interesting recent contribution: separating long-horizon plan state from per-step progress state lets the orchestrator notice when it's stuck and re-plan, which is exactly the failure mode that kills naive autonomous agents. And **Blackboard** is older than LLMs by decades; it's quietly becoming the default coordination substrate again because writing to shared state is just easier than passing fully-formed messages.

## Industry Examples By Topology

Patterns are easier to evaluate against concrete workloads. The table below pairs each topology with one example from healthcare/life-sciences (HCLS) and one from manufacturing — two domains where multi-agent systems are already in production and where the failure costs are high enough that the topology choice actually matters.

| Topology | HCLS example | Manufacturing example |
|---|---|---|
| Central decomposer | Prior-auth processing: coordinator delegates to eligibility, medical-necessity, formulary specialists | Root-cause analysis on a quality incident: orchestrator delegates to process-data, maintenance-log, supplier-history agents |
| Fixed ordered | Claims adjudication: extract → ICD/CPT coding → policy check → pricing → pay | Quality inspection: image capture → defect detection → classification → disposition |
| Peer handoff | Patient triage: intake → specialty (cardio/onco) → discharge-planning agent, context transferred | Field service: diagnostic agent → parts-lookup agent → service-dispatch agent |
| Concurrent | Literature review fanned out across PubMed, ClinicalTrials.gov, FDA label DB, then aggregated | Multi-sensor anomaly detection: parallel detectors on vibration, temperature, current, vote on fault |
| Critique loop | Medical coding with CDI generator + auditor loop against source documentation | CAD/part design: generator proposes geometry, FEA+cost evaluator critiques, iterate to spec |
| Shared thread | Virtual tumor board: oncologist + radiologist + pathologist + surgeon agents discuss a case | Engineering change review: design + process + quality + supply-chain agents review an ECN |

The pattern that recurs across both columns is that *fixed-ordered* and *central-decomposer* shapes dominate the highly-regulated, audit-heavy workflows (claims, prior-auth, quality inspection), while *shared-thread* and *critique-loop* shapes appear wherever the work is genuinely deliberative and the value is in the disagreement between specialists.

## Collaboration Mode (Orthogonal Axis)

Topology answers *who talks to whom*. It doesn't answer *why*. Collaboration mode is the orthogonal axis: it describes the incentive structure between the agents, which can be mixed independently of the graph shape.

- **Cooperation** — agents share a goal.
- **Competition** — adversarial (e.g., debate).
- **Coopetition** — mixed, e.g., negotiation.

This axis matters because adversarial setups (debate, red-team/blue-team) demonstrably improve factuality on reasoning tasks even when the topology is a simple shared thread — the gains come from the incentive, not the wiring. See survey: [Tran et al., "Multi-Agent Collaboration Mechanisms: A Survey of LLMs," arXiv:2501.06322](https://arxiv.org/abs/2501.06322).

## Choosing a Pattern — Quick Heuristics

Most pattern-selection mistakes come from reaching for the most autonomous shape first. The heuristics below are roughly ordered from least to most autonomy, and the right answer is usually the first one that fits.

- Subtasks known & ordered → **Prompt Chaining / Sequential**
- Distinct input classes → **Routing**
- Independent work that can fan out → **Parallelization (Sectioning)**
- Need higher reliability on same task → **Parallelization (Voting) / Debate**
- Subtasks unknown at plan time → **Orchestrator–Workers / Supervisor**
- Many (>~8) specialists → **Hierarchical tree** ([LangGraph guidance](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/hierarchical_agent_teams/))
- Open-ended dialogue, peer expertise → **Group Chat / Swarm**
- Quality iterable via critique → **Evaluator–Optimizer**
- Long-horizon with drift → **Magentic-One dual-loop** (re-plan on stall)

## Common Tradeoffs

The patterns above are not free. Each step toward more autonomy buys flexibility at a measurable cost in tokens, latency, and debuggability. A few tradeoffs are universal enough to be worth calling out explicitly.

- More autonomy → higher token cost, more failure modes, harder to debug. Anthropic explicitly recommends starting with the simplest workflow; add agents only when needed. ([Anthropic](https://www.anthropic.com/engineering/building-effective-agents))
- Supervisor accuracy degrades as tool/agent count grows → decompose hierarchically. ([LangGraph Hierarchical](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/hierarchical_agent_teams/))
- Swarm/peer graphs are flexible but state-transfer and termination are hard; handoffs must preserve context.
- Multi-agent systems are worth it when the domain is genuinely cross-functional; otherwise a single agent with good tools usually wins.

The last point is the one most teams underweight. A capable single agent with a well-curated tool belt frequently beats a multi-agent system on the same task, because every additional agent adds a translation boundary where context can be lost. The justification for going multi-agent should come from the domain, not from the architecture being interesting.

---

# Part II — Multi-Agent Orchestration in Snowflake, Today

The agent space repeats the same pitch a hundred times: *"the future is multi-agent."* Specialized agents, talking to each other, dividing labor, reasoning in parallel. It's a compelling story. It's also, for most teams, the wrong story to start with.

The interesting question isn't whether multi-agent systems are coming. They are. The interesting question is what can actually be built *today* — with primitives that exist, on a platform already trusted with sensitive data — and whether that gets to 80% of the value without 80% of the operational cost.

For Snowflake customers, the answer is yes. There's a small, learnable catalog of patterns that compose into real multi-agent workloads inside Snowflake right now, with no roadmap dependencies and no PHI leaving the perimeter. What follows is a walk through each pattern, and then a real healthcare scenario that shows how they fit together.

The six patterns map cleanly onto the [topology catalog](#cross-framework-alias-table) from Part I:

| Snowflake Pattern | Topology equivalent |
|---|---|
| [Pattern 1: One Cortex Agent, Many Tools](#pattern-1-one-cortex-agent-many-tools) | Central decomposer / Agents-as-Tools |
| [Pattern 2: Two agents, stored-proc bridge](#pattern-2-when-two-agents-are-genuinely-required) | Manager-style cross-agent call |
| [Pattern 3: Streams + Tasks pipeline](#pattern-3-when-the-workflow-is-the-orchestrator) | Fixed ordered / Prompt Chaining |
| [Pattern 4: Framework in containers](#pattern-4-when-a-real-framework-is-required) | Whatever the framework expresses (graph, hierarchical, etc.) |
| [Pattern 5: Streamlit/Native App as orchestrator](#pattern-5-when-the-human-is-part-of-the-orchestration) | Human-in-the-loop, often with shared thread |
| [Pattern 6: AISQL row-by-row](#pattern-6-when-scale-is-being-mistaken-for-complexity) | Concurrent / Sectioning at table scale |

## Pattern 1: One Cortex Agent, Many Tools

Most production "multi-agent" systems aren't really multi-agent. They're one orchestrator-shaped agent with a bag of specialized tools — and that is usually the right answer.

A Cortex Agent is, at its core, a planner: it reads the user's question, decides which tools to call in what order, looks at the results, and decides whether to call more tools or answer. Giving it Cortex Analyst for structured queries, Cortex Search for unstructured retrieval, custom UDFs for business logic, MCP servers for external systems, and stored procedures for actions effectively produces a small mesh of specialists with a single intelligent router on top.

```
                 ┌──────────────────────────┐
                 │   Cortex Agent (router)  │
                 │   plan → call → reflect  │
                 └────┬───┬────┬────┬───────┘
                      │   │    │    │
        ┌─────────────┘   │    │    └────────────┐
        ▼                 ▼    ▼                 ▼
 Cortex Analyst    Cortex Search   Stored Proc /     MCP Server
 (semantic view)   (RAG over docs) UDF / SP Python   (external tools)
```

This pattern works because the planner *is* the orchestrator. There's no need for a separate framework, a separate runtime, or a separate set of governance controls. Everything happens inside one agent object, with one RBAC story, and one audit trail. This is the recommended starting point. The vast majority of Cortex Agent deployments in production today never need to graduate beyond it.

**Memory.** Conversational memory comes free via Cortex Agent **Threads** — a thread ID persists turn-by-turn context for the duration of an interaction without any client-side state management. Anything more durable (per-user preferences, longitudinal context, learned facts) gets modeled as a Snowflake table that the agent reads and writes through a tool. Keep Threads for the conversation; keep tables for the long-term memory. (See [A Note on Memory](#a-note-on-memory).)

**External tools via MCP Connectors.** The "MCP Server" branch in the diagram above is now a first-class capability. Snowflake's **MCP Connectors** ([docs](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp-connectors)) let a Cortex Agent consume tools hosted on remote MCP servers — Atlassian (Jira/Confluence), GitHub, Glean, Linear, Salesforce out of the box, plus any custom MCP server behind an OAuth endpoint. The setup is `CREATE API INTEGRATION` (with `OAUTH_DYNAMIC_CLIENT` or standard OAuth credentials) followed by `CREATE EXTERNAL MCP SERVER`, then a reference in the agent spec. At runtime the agent calls `tools/list` to discover the remote tools and `tools/call` to invoke them; OAuth flows authenticate end users to the third-party service. This closes a large chunk of what previously required custom external-function plumbing or A2A-style wrappers — for tool-shaped interop with third-party SaaS, MCP Connectors are now the answer. Caveat worth naming: Snowflake explicitly does not warrant or vet external MCP servers, so trust, data rights, and compliance with the third party's terms remain the customer's responsibility.

## Pattern 2: When Two Agents Are Genuinely Required

Sometimes one agent isn't enough — not because of capability, but because of *ownership*. The medical-policy team owns the agent that reasons about clinical evidence. The ops team owns the agent that takes write actions on the EHR. They have different release cadences, different access boundaries, different blast radii. Merging them into a single agent collapses governance distinctions that the organization specifically created for a reason.

The pragmatic way to wire two Cortex Agents together today is a stored procedure. Agent A calls a tool; the tool is a stored proc that holds the credentials and role for Agent B; the proc invokes Agent B's `agent:run` API and returns the result. Agent A doesn't know or care that the thing on the other end is also a Cortex Agent.

```
  Agent A  ──tool call──▶  Stored Proc  ──REST/SQL──▶  Agent B
                              │
                              ▼
                       Calls agent:run on Agent B,
                       returns the result back to Agent A
```

It is admittedly inelegant. It is bespoke per integration, doesn't scale to dozens of agents across vendors, and there's no agent-card discovery happening. But it works, it preserves governance boundaries, and it's the closest approximation of "agent-to-agent" available in Snowflake today without reaching for [an A2A wrapper](#part-iii--the-a2a-wrapper-when-snowflake-has-to-talk-to-the-outside-mesh).

**Memory.** This is the most common source of bugs in this pattern: each agent has its own thread, and **memory does not cross the bridge** unless explicitly passed. Whatever Agent A "knows" stays in Agent A's thread; Agent B starts fresh on every call. Two practical fixes — pass the relevant context as arguments through the stored proc, or designate a shared Snowflake memory table that both agents read and write as a tool. The shared-table approach scales better when the conversation needs to outlive any single agent invocation.

## Pattern 3: When the Workflow Is the Orchestrator

There's a class of problem where the agent isn't supposed to plan. The shape is a fixed sequence of steps, applied repeatably, every night, to every new record — and the audit trail must be deterministic. Adjudication. Risk scoring. Coding. Anything regulated.

Here, the orchestrator is the data pipeline. A Stream watches an input table; a Task fires when there's new data and calls Agent A; the result lands in a staging table; a second Task picks that up and calls Agent B; and so on. Replacing the staging tables with Dynamic Tables yields declarative materialization between steps for free.

```
   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
   │  Source  │───▶│  Stream  │───▶│  Task 1  │───▶│ Stage A  │
   │  Table   │    │  (CDC)   │    │ Agent A  │    │  Table   │
   └──────────┘    └──────────┘    └──────────┘    └────┬─────┘
                                                        │
                          ┌─────────────────────────────┘
                          ▼
                    ┌──────────┐    ┌──────────┐    ┌──────────┐
                    │  Stream  │───▶│  Task 2  │───▶│ Stage B  │
                    │          │    │ Agent B  │    │  Table   │
                    └──────────┘    └──────────┘    └────┬─────┘
                                                         │
                          ┌──────────────────────────────┘
                          ▼
                    ┌──────────┐    ┌──────────┐    ┌──────────────┐
                    │  Stream  │───▶│  Task 3  │───▶│   Final      │
                    │          │    │ Agent C  │    │   Decisions  │
                    └──────────┘    └──────────┘    └──────────────┘
```

The trade-off is intentional: the agent's autonomy to decide who calls whom is given up in exchange for repeatability and lineage. For batch enrichment flows where compliance matters more than creativity, that is the right exchange to make.

**Memory.** There is no conversation, so there is no conversational memory to manage. What looks like memory in this pattern is actually **state in tables** — the staging table or Dynamic Table between Task N and Task N+1 *is* the memory. The discipline that follows is different: idempotency, replay safety, schema evolution, retention policies. Time Travel and zero-copy clones make replay and "what did the pipeline see yesterday" straightforward operations, which is one of the underappreciated reasons workflow-driven orchestration is so durable.

## Pattern 4: When a Real Framework Is Required

Eventually a workload appears where neither a single Cortex Agent nor a fixed pipeline is enough. The shape calls for stateful graphs, retries with branching, human-in-the-loop checkpoints, dynamic fan-out — the kind of thing LangGraph, CrewAI, and Semantic Kernel were designed for.

Leaving Snowflake isn't required to use them. Snowpark Container Services runs any of these frameworks as a container *inside* the Snowflake security perimeter. The graph nodes call Cortex Agents and AI functions like any other tool; outbound traffic to external APIs goes through external access integrations. PHI never leaves the perimeter; the framework's complexity stays where it's needed.

```
            ┌────────────────────────────────────────────┐
            │      Snowpark Container Services           │
            │  ┌──────────────────────────────────────┐  │
            │  │  LangGraph / CrewAI orchestrator     │  │
            │  │   nodes call:                        │  │
            │  │     • Cortex Agent A                 │  │
            │  │     • Cortex Agent B                 │  │
            │  │     • Cortex AI functions            │  │
            │  │     • External APIs                  │  │
            │  └──────────────────────────────────────┘  │
            └────────────────────────────────────────────┘
```

This is the most powerful pattern available today. It is also the most operationally heavy. It is appropriate when there is genuine graph complexity to express — and not before.

**Memory.** Whatever the framework provides — LangGraph checkpointers, CrewAI memory modules, Semantic Kernel state. The important detail is that container memory is **ephemeral**: a restart, a redeploy, or an autoscale event wipes it. For anything durable, persist checkpoints to a Snowflake table or stage and restore on container start. Long-term semantic memory plugs in cleanly to Cortex Search (vector retrieval) backed by Snowflake-governed tables, which gives the framework a memory store that inherits Snowflake's RBAC and lineage.

## Pattern 5: When the Human Is Part of the Orchestration

Some workflows aren't autonomous. They're cooperative. A case manager is sitting in front of a screen, looking at a patient, asking questions, taking actions. The "orchestrator" in that scenario is the application — Streamlit-in-Snowflake or a Native App — that holds the conversation state, decides which agents to call when, and gives the human a place to approve, edit, or escalate.

This pattern is the fastest to ship. It is also the most tactical: the orchestration logic lives in app code and doesn't reuse outside the app. That is acceptable when the app *is* the product. It becomes a problem when the goal was a reusable agent platform and what got built was a Streamlit prototype that happens to talk to three agents.

**Memory.** The pattern naturally splits into three layers. The live session uses `st.session_state` for UI-local state. Each agent the app talks to keeps its own Cortex Agent thread for turn-by-turn context. Anything that should outlive the session (user preferences, prior decisions, curated facts) lands in a Snowflake table the app reads on login and writes when the user takes an action. The app is in the unique position of being able to *unify* memory across agents — it sees all of them and can stitch their threads together for the human.

## Pattern 6: When Scale Is Being Mistaken for Complexity

Here is the pattern that quietly handles more "multi-agent" use cases than any of the others: Cortex AISQL. `AI_CLASSIFY`, `AI_FILTER`, `AI_AGG`, `AI_SUMMARIZE_AGG`, `AI_COMPLETE` — applied row-by-row to a table, composed as CTEs, executed as SQL.

It is not orchestration in the agentic sense. There is no planner, no conversation, no peer agents. But when someone says "classify every ticket, then summarize each cluster, then route to a queue," what they often actually need is three SQL steps over a table, not three agents talking to each other. AISQL is cheaper, simpler, more governable, and parallelizes across millions of rows for free. Before reaching for any of the patterns above, the first question worth asking is whether the problem is genuinely "agents are needed" or "bulk LLM reasoning over rows is needed." If it is the latter, AISQL is the right tool, and a great deal of orchestration work disappears.

**Memory.** Stateless by design — each row is independent, which is precisely what allows the pattern to parallelize across millions of rows. The "memory" is the table itself: chain CTEs to carry context from one step to the next, materialize intermediate results as tables when context needs to outlive a single query, and use temporal columns when the model needs to see a row's history. There is no thread, no checkpoint, nothing to manage — and that is the design intent.

## Composition: 30-Day Readmission Prevention

Patterns are easier to understand in service of a real problem. Consider one that every payer and integrated delivery network in the United States cares about: **predicting and preventing 30-day hospital readmissions.**

The financial pressure is direct. Under CMS's Hospital Readmissions Reduction Program, hospitals can lose up to 3% of their Medicare reimbursement for excess readmissions. A single avoided readmission saves $10,000 to $25,000. The clinical pressure is more direct still: patients who bounce back to the hospital are usually sicker, scared, and underserved by the discharge process.

For every discharge, the system needs to score the patient's readmission risk, explain *why* they're high-risk in language a case manager can act on, recommend interventions, hand off to a case manager with a ranked worklist, and close the loop by feeding outcomes back into the model. No single LLM call can do that responsibly. No single agent can either. The patterns get composed.

```
   ┌──────────────────────────────────────────────────────────────────┐
   │  NIGHTLY BATCH (Pattern 3 + Pattern 6)                           │
   │                                                                  │
   │  Discharges  ─Stream─▶  Task: AISQL screen  ─▶ Risk-scored cohort│
   │  (EHR feed)             AI_CLASSIFY +                            │
   │                         AI_COMPLETE per row                      │
   │                                                                  │
   │            ─▶  Task: enrich w/ claims, labs, SDOH  ─▶  Ranked    │
   │                Dynamic Table                            Worklist │
   └────────────────────────────────────┬─────────────────────────────┘
                                        │
                                        ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  CASE MANAGER UI (Pattern 5: Streamlit-in-Snowflake)             │
   │                                                                  │
   │     Worklist  ──opens patient──▶  Conversational pane            │
   │                                       │                          │
   │                                       ▼                          │
   │                              ┌──────────────────┐                │
   │                              │  Cortex Agent    │ (Pattern 1)    │
   │                              │   "Care Coach"   │                │
   │                              └─────┬─────┬──────┘                │
   │                                    │     │                       │
   │              ┌─────────────────────┘     └─────────────────┐     │
   │              ▼                                              ▼    │
   │      Cortex Analyst                                  Cortex Search│
   │  (semantic view over                              (chart notes,  │
   │   claims, labs, meds)                              discharge sum) │
   │              │                                              │    │
   │              └──────────────────┬──────────────────────────┘    │
   │                                 ▼                                │
   │                          Custom tools (UDFs):                    │
   │                          • Care-gap lookup                       │
   │                          • Med-reconciliation check              │
   │                          • SDOH risk signals                     │
   │                          • Schedule TCM visit (action)           │
   └──────────────────────────────────┬───────────────────────────────┘
                                      │
                                      ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  CROSS-AGENT HANDOFF (Pattern 2: stored proc bridge)             │
   │                                                                  │
   │  Care Coach Agent  ──tool: schedule_intervention──▶              │
   │       Stored Proc  ──agent:run──▶  Scheduling Agent              │
   │                                    (separate schema, ops team)   │
   │                                    books TCM visit, posts to EHR │
   └──────────────────────────────────────────────────────────────────┘
```

The story unfolds in three acts.

**The night before.** Every discharge from the previous day flows in via a Stream. A Task wakes up and runs an AISQL pipeline that classifies each discharge by primary condition family, extracts structured risk signals from the unstructured discharge summary (medication non-adherence, social isolation, missing PCP follow-up), and emits a calibrated risk score. This is [Pattern 6](#pattern-6-when-scale-is-being-mistaken-for-complexity) carrying the load: thousands of discharges processed in minutes, no orchestration overhead, just SQL.

A second Task — [Pattern 3](#pattern-3-when-the-workflow-is-the-orchestrator) — picks up the scored cohort and joins it with claims data (prior utilization), labs (recent abnormals), medications (polypharmacy flags), and SDOH attributes (housing, transportation, language). The output is a Dynamic Table named `READMIT_WORKLIST`. It's always fresh, always deterministic, always auditable.

**The morning of.** A case manager opens a Streamlit app. The worklist is already there, ranked by risk and need. She clicks the highest-risk patient and a conversational pane opens scoped to that member. This is [Pattern 5](#pattern-5-when-the-human-is-part-of-the-orchestration): the human is in the loop, and the app is the orchestrator.

She asks, "Why is she high-risk?" The "Care Coach" Cortex Agent — [Pattern 1](#pattern-1-one-cortex-agent-many-tools) — plans across its tools. Cortex Analyst pulls structured signals from a semantic view of claims, labs, and meds. Cortex Search retrieves passages from the discharge summary and recent chart notes. Custom UDFs surface care gaps and med-reconciliation issues. The agent composes an answer with citations, all inside the Snowflake perimeter.

**The action.** The case manager decides to schedule a transitional care visit. A button click invokes the agent's `schedule_intervention` tool. Under the hood, that tool is a stored procedure — [Pattern 2](#pattern-2-when-two-agents-are-genuinely-required) — that calls a separate Cortex Agent owned by the operations team. That second agent has its own schema, its own RBAC, write privileges to the EHR through an external access integration. The TCM visit gets booked, the EHR gets updated, and a result row gets written back to a results table.

Overnight, the loop closes. A Task feeds the day's interventions and outcomes back into the model retraining pipeline. The next morning's worklist reflects yesterday's care.

What appears above is four patterns composed into one production system. None of them, alone, would suffice. Pattern 1 can't do nightly batch over thousands of rows efficiently. Pattern 6 can't hold a conversation. Pattern 3 can't reason about an individual patient's open-ended questions. Pattern 5 is just a UI without the agents and pipelines underneath. Pattern 2 is a bridge, not a system. Together, they're an end-to-end care management platform that exists today, runs on primitives that exist today, and never lets PHI leave the Snowflake perimeter.

What isn't required is also worth naming. There's no need for an A2A wrapper, because every component lives in Snowflake. There's no need for an external orchestrator, because Tasks plus the Cortex Agent planner cover it. There's no need for a separate vector database, because Cortex Search is built in. There's no need for an external LLM gateway, because Cortex AI functions are governed and metered inside the platform.

## A Note on Memory

Memory in agent systems isn't one thing. It's three.

**Conversational memory** is the immediate turn-by-turn context — what was just said, what's being referenced, what the agent is currently working on. Cortex Agent **Threads** handle this natively: a thread ID, and the platform keeps the context for the duration of the interaction.

**Episodic memory** is what the system has learned across sessions — preferences, prior decisions, curated facts about a user or entity. There's no native primitive for this; the right shape is a Snowflake table the agent reads and writes through a tool. It inherits RBAC, masking, and lineage automatically. In multi-tenant systems, scope it with row-access policies bound to immutable session attributes — the same mechanism the Cortex Agents multi-tenancy guide uses for data.

**Semantic / long-term memory** is the durable, often-curated knowledge layer — a longitudinal care plan, a customer's history, an organization's policy precedents. Cortex Search is the right home: the embedding index lives in Snowflake, the source documents stay governed, and retrieval is just another tool the agent calls.

A few cross-cutting principles fall out of this:

- **Threads are short-lived; tables are durable.** Don't expect threads to do persistent memory's job. Use them for what they are good at and write durable state to a table.
- **Memory does not cross agent boundaries automatically.** When two agents need to share context ([Pattern 2](#pattern-2-when-two-agents-are-genuinely-required), parts of [Pattern 5](#pattern-5-when-the-human-is-part-of-the-orchestration)), the memory has to live in a place both agents can read — a shared table, not either agent's thread.
- **Memory is data, and Snowflake's governance applies to it.** PHI in a memory table is still PHI. Row-access policies, masking, retention rules — all the same.
- **State is not memory.** [Pattern 3](#pattern-3-when-the-workflow-is-the-orchestrator) has state in tables but no memory in the agent sense. The discipline is different (idempotency, replay) and conflating the two leads to bad designs.

The practical recipe for most production systems ends up being a mix: Threads for conversation, one or two well-modeled memory tables for episodic facts, Cortex Search for semantic recall. Build only the layers the workload actually needs.

## So Which Pattern Fits?

The accurate answer is: usually more than one. But starting from scratch, the right question is what shape the problem actually has.

A smart router that picks the right specialist for an open-ended question maps to [Pattern 1](#pattern-1-one-cortex-agent-many-tools). Two agents owned by two teams whose governance shouldn't be merged map to [Pattern 2](#pattern-2-when-two-agents-are-genuinely-required) as the bridge. A fixed workflow that needs batch determinism maps cleanly to [Pattern 3](#pattern-3-when-the-workflow-is-the-orchestrator). Real graph complexity — branching, retries, human-in-the-loop nodes, fan-out — graduates to [Pattern 4](#pattern-4-when-a-real-framework-is-required) inside Snowpark Container Services. A human-driven workflow with multiple agents fits [Pattern 5](#pattern-5-when-the-human-is-part-of-the-orchestration) and ships fastest. And the same reasoning step over millions of rows isn't a multi-agent problem at all; it is [Pattern 6](#pattern-6-when-scale-is-being-mistaken-for-complexity).

The patterns are not in conflict. They are in composition. The teams that get the most value out of Snowflake's agent stack today are the ones that learned to recognize which pattern fits which part of the workload, and stopped attempting to force everything through a single one.

---

# Part III — The A2A Wrapper: When Snowflake Has To Talk To The Outside Mesh

For all of the above, there's a real gap worth naming: Snowflake doesn't natively speak agent-to-agent across organizational and vendor boundaries. There's no peer protocol for agents to discover each other. There's no native long-running task lifecycle that spans systems. [Pattern 2](#pattern-2-when-two-agents-are-genuinely-required)'s stored-proc bridge handles two-agent integrations, but it's bespoke and it doesn't compose into a cross-vendor mesh.

That said, the gap is narrower than it was. **MCP Connectors** ([docs](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp-connectors)) now let a Cortex Agent consume external MCP servers as tools, and `CREATE MCP SERVER` ([docs](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp)) lets Snowflake objects (including Cortex Agents themselves) be exposed as MCP tools to external clients. Together, these cover a meaningful slice of what teams previously reached for A2A to solve — anywhere the interop is *tool-shaped* (one side calls, the other returns), MCP is the right answer today.

A2A still wins where the interop is genuinely *peer-to-peer*: long-running tasks with their own lifecycle, agent cards advertising capabilities for discovery, symmetric delegation between two equal agents that both reason and plan. For a payer whose Cortex Agent needs to participate in a workflow with a PBM's agent and a care vendor's agent, all running on different platforms — A2A is the right answer, and a thin wrapper around `agent:run` is the practical way to get there.

## High-Level Architecture

```
                ┌─────────────────────────────────────────┐
                │        EXTERNAL AGENT ECOSYSTEM         │
                │  (other vendors, partners, frameworks)  │
                │                                         │
                │   Salesforce    Google     LangGraph    │
                │     Agent       Agent        Agent      │
                │       │           │            │        │
                └───────┼───────────┼────────────┼────────┘
                        │           │            │
                        │     A2A protocol       │
                        │   (open, standard)     │
                        ▼           ▼            ▼
                ┌─────────────────────────────────────────┐
                │           A2A WRAPPER LAYER             │
                │   • Publishes Agent Card (capabilities) │
                │   • Translates A2A ⇄ Snowflake API      │
                │   • Auth bridge                         │
                └────────────────────┬────────────────────┘
                                     │
                                     ▼
                ┌─────────────────────────────────────────┐
                │            SNOWFLAKE PLATFORM           │
                │                                         │
                │              Cortex Agent               │
                │             ┌─────┴─────┐               │
                │      Cortex Analyst  Cortex Search      │
                │             │             │             │
                │       Structured     Unstructured       │
                │          data            data           │
                │                                         │
                │     Governance · RBAC · Lineage         │
                └─────────────────────────────────────────┘
```

## Why This Matters

- **Discoverability** — outside agents find your Snowflake agent the same way they find any other A2A agent (read the Agent Card, done).
- **No bespoke integrations** — one wrapper instead of N custom connectors per partner/tool.
- **Composition** — your Cortex Agent becomes a *participant* in multi-agent workflows orchestrated elsewhere (e.g., a Salesforce agent delegates "give me Q3 revenue analysis" to your Snowflake agent).
- **Symmetry** — your Cortex Agent can also *call out* to other A2A agents as collaborators, not just be called.
- **Governance stays put** — data never leaves Snowflake's perimeter; the wrapper only translates the conversation, so RBAC, masking, and lineage still apply.
- **Future-proofing** — A2A is becoming the lingua franca for cross-vendor agents. A wrapper means you don't get locked out of that ecosystem while waiting for native support.

The mental model: think of the wrapper as a **diplomatic translator**. Snowflake speaks its native dialect (Cortex Agents API), the outside world speaks A2A, and the wrapper lets them negotiate without either side changing.

## HCLS Worked Example: Prior Authorization

### The Business Scenario

A large payer is processing a prior authorization (PA) request for a high-cost oncology therapy. Getting to a decision requires four very different agents, owned by four different teams, running in four different places:

| Agent | Owner | Lives in | Job |
|---|---|---|---|
| **Intake Agent** | Care management vendor | Salesforce Health Cloud | Receives the PA request, validates member eligibility, gathers clinical context |
| **Clinical Evidence Agent** | Payer's medical policy team | Snowflake (Cortex Agent) | Reads the patient's structured claims/labs and unstructured chart notes; checks the request against medical-policy criteria |
| **Drug Pricing & Network Agent** | PBM partner | External SaaS (LangGraph) | Validates formulary tier, channel, network pharmacy, rebate eligibility |
| **Decision & Letter Agent** | Internal ops | Snowflake (Cortex Agent, separate account/schema) | Composes the approval/denial letter, files it with the provider portal, triggers downstream notifications |

### Conversation Flow

```
   ┌──────────────┐
   │ Intake Agent │  PA request submitted by provider
   │ (Salesforce) │
   └──────┬───────┘
          │  A2A: "Run policy review for member M, drug D, dx C50.911"
          ▼
   ┌─────────────────────────────────────────┐
   │           A2A WRAPPER LAYER             │
   │  Translates request → agent:run call    │
   │  Stamps tenant/region session attrs     │
   └──────────────────┬──────────────────────┘
                      │
                      ▼
   ┌─────────────────────────────────────────┐
   │     Clinical Evidence Cortex Agent      │
   │  Cortex Analyst → claims, labs, dx      │
   │  Cortex Search  → progress notes, path  │
   │  Returns: "Meets policy 4.7.A; cite..." │
   └──────────────────┬──────────────────────┘
                      │  A2A: needs pricing check
                      ▼
   ┌──────────────────────────┐
   │ PBM Pricing Agent        │  Confirms formulary, network, $$
   │ (LangGraph)              │
   └──────────────┬───────────┘
                  │  A2A: hand off package
                  ▼
   ┌──────────────────────────┐
   │ Decision & Letter Agent  │  Drafts letter, files PA, notifies
   │ (Snowflake Cortex Agent) │  • LLM via Cortex AI functions
   │                          │  • Letter template via Cortex Search
   │                          │  • Filing/notify via stored proc +
   │                          │    external access integration
   └──────────────────────────┘
```

The whole thing happens as one coherent multi-agent conversation. **No human-built point-to-point integrations.** Each agent only had to learn one protocol (A2A) and publish one Agent Card.

### What the Wrapper Specifically Buys You

- **PHI never leaves Snowflake.** The Clinical Evidence Agent does the chart/claims reasoning *inside* the governed perimeter. Only the policy decision and citations cross the wire.
- **Each partner stays in their stack.** The PBM doesn't have to onboard into Snowflake; the payer doesn't have to push PHI into Salesforce.
- **Auditability.** The wrapper logs every A2A turn, and Snowflake's lineage logs every query the Cortex Agent ran — so the appeal/audit trail is complete on both sides.
- **Plug-and-play partners.** When the payer adds a second PBM next year, that PBM just publishes an Agent Card. No new integration project.

### Is "Decision & Letter Agent on Snowflake" Realistic?

**Yes, and arguably preferable.** Everything that agent needs is already in the platform:

- **LLM authoring** — Cortex AI functions (`AI_COMPLETE`, `AI_SUMMARIZE_AGG`) generate the approval/denial letter body using approved templates and the citations passed in from the Clinical Evidence Agent.
- **Template + precedent retrieval** — a Cortex Search service over prior letters, regulatory templates (CMS, state DOI), and member-communication policies.
- **Action execution** — stored procedures + external access integration handle (a) filing the PA decision into the provider portal, (b) posting to the member app, (c) firing notifications to the case manager via a Notification Integration.
- **Governance** — the same RBAC, masking, and row-access policies that protect PHI for the Clinical Evidence Agent now also protect the letter content and member contact info. One governance model, not two.

**Two practical reasons it lives in a separate Snowflake account/schema** (rather than just being a tool on the Clinical Evidence Agent):

1. **Different team, different lifecycle.** The medical-policy team owns clinical reasoning; the ops team owns member communications, regulatory letter language, and downstream filing. Separate agent objects = separate ownership, deployment cadence, and access boundaries.
2. **Different blast radius.** The Decision Agent has *write* side-effects (files PA, sends notifications). Keeping it in its own account/role isolates those external-side-effect privileges from the read-heavy clinical agent.

### Then Why Still Wrap It in A2A If Both Endpoints Are Snowflake?

Because the value of A2A isn't "Snowflake ↔ non-Snowflake" — it's **a uniform protocol for any agent-to-agent hop in the mesh.**

- The orchestrator (or the Clinical Evidence Agent calling forward) shouldn't need to know whether the next hop is Snowflake, Vertex, LangGraph, or a partner SaaS. It just calls A2A.
- If tomorrow the ops team migrates the Decision Agent to a different platform (or a vendor offers a better letter-authoring agent), nothing upstream changes.
- A2A gives you a **single observability surface** across all hops — same task IDs, same lifecycle events, same audit envelope — even when both ends happen to be Snowflake.
- It avoids the trap of "Snowflake-to-Snowflake special case" code paths that drift over time.

In short: putting the Decision Agent in Snowflake is the right architectural call (governance, cost, proximity to data). Wrapping it in A2A is the right *interface* call (uniform mesh, future-proof, swappable).

## Why We Cannot Do This Inside Snowflake Today

The wrapper is a workaround, not a preference. It exists because Snowflake (as of today) is **a great agent runtime, but not yet an A2A *server*.** Specifically:

1. **No Agent Card endpoint.** A2A discovery requires each agent to publish a JSON Agent Card at a well-known URL. Snowflake's Cortex Agents are addressable via the proprietary `agent:run` REST API — there is no `/.well-known/agent.json` exposed by the platform.
2. **No A2A method surface.** A2A speaks JSON-RPC methods like `message/send`, `tasks/get`, `tasks/cancel`, with a defined task lifecycle (`submitted → working → completed/failed`) and SSE streaming envelope. Cortex Agents return their own event stream shape. Until Snowflake natively maps `agent:run` to the A2A method set, an outside agent literally cannot talk to it.
3. **Authentication mismatch.** A2A clients expect the auth scheme advertised in the Agent Card (typically OAuth bearer tokens issued by the agent's own IdP). Snowflake auth is PAT / key-pair JWT / Snowflake OAuth — none of which a generic A2A client knows how to negotiate. The wrapper performs the **token exchange** between an external IdP token and a Snowflake credential.
4. **No outbound A2A client.** A Cortex Agent today can call **MCP tools** and custom UDFs/stored procs, but it cannot natively invoke another A2A agent as a tool. Symmetric multi-agent orchestration (Snowflake delegating *out* to a peer agent) requires a wrapper-side client.
5. **Streaming + long-running tasks.** A2A standardizes long-running tasks with SSE updates and resumable task IDs. Cortex Agent streaming uses Snowflake's own event format. The wrapper bridges the two so an external orchestrator can poll/stream task state in the way it expects.
6. **Identity propagation across the mesh.** In a multi-agent workflow, the *originating* user identity needs to flow through every hop so each platform can apply its own RBAC/row-access. Today there is no native way for an inbound A2A call to map to a Snowflake session attribute. The wrapper is where that mapping is expressed (and where multi-tenant immutable session attributes get stamped on the call).

## The Forward Look

Most of the above are translation problems, not architectural ones. A future Snowflake release that exposes Cortex Agents over A2A natively would let you delete the wrapper and point external agents directly at the Snowflake endpoint. Until then, the wrapper is the **smallest, lowest-risk way to participate in the cross-vendor agent mesh without compromising governance or waiting on roadmap.**

The story of multi-agent orchestration in Snowflake today isn't really a story about agents at all. It is a story about composition: about recognizing that the platform already provides the primitives — planners, tools, pipelines, containers, AISQL, Streamlit — and that the work is not building a multi-agent framework from scratch, but combining what is there into something the business actually needs. In most cases, that is sufficient. When it is not, the door to A2A remains open.
