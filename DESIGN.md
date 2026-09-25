# Agent Graph — Design

A framework for building agentic systems as a graph. The graph *is* the app:
orchestration, tools, sub-agents, checks, and guardrails are nodes. Edges say
who talks to whom and under what rule. The UI lets you build, edit, run, and
watch the graph.

## 1. Node types

| Node | Metaphor | Holds | Execution tag |
|---|---|---|---|
| **King** | Citadel | Orchestrator: system prompt or router code, loop budget, final-answer rule | `orchestrator` — loop starts and ends here. Exactly one per graph. |
| **Ammunition** | Armory | Tool: name, description, JSON input schema, output schema, code | `tool` — pure function, no LLM |
| **Knight** | Army | Sub-agent: system prompt, output schema, optional code, model, tool allowlist | `agent` — has its own mini loop |
| **Advisor** | Council | Validator / reflection / SME: prompt **or** code, pass rule, max retries | `check` — returns `pass` or `feedback` |
| **Warden** | Court | Guardrail: rules (prompt or regex/code), action on hit (block / rewrite / warn) | `gate` — runs before input and before final output |

Every node has: `id`, `kind`, `name`, `config` (the fields above), `position`.

## 2. Edge types

Edges are typed. The type decides the runtime behaviour, not free text.

| Edge | From → To | Meaning | Direction |
|---|---|---|---|
| `delegate` | King → Knight | Give a task. Carries a task template. | one-way, Knight returns a result |
| `use` | Knight/King → Ammunition | May call this tool | one-way |
| `check` | Knight/King → Advisor | Send output for validation. Advisor sends `feedback` back until `pass` or `max_retries`. | loop |
| `gate` | Warden → King | Runs on user input (`input` gate) or on the final answer (`output` gate) | one-way |
| `handoff` | Knight → Knight | Pass a result to another agent directly (no return) | one-way |

Edge fields: `id`, `type`, `from`, `to`, `label`, `config` (task template, retry cap, gate stage, etc.).

## 3. Graph file (single source of truth)

```json
{
  "name": "Company memo agent",
  "nodes": [
    { "id": "king", "kind": "king", "name": "Orchestrator",
      "config": { "prompt": "...", "maxTurns": 12 } },
    { "id": "research", "kind": "knight", "name": "Research agent",
      "config": { "prompt": "...", "outputSchema": {"...": "..."}, "model": "claude-sonnet-5" } },
    { "id": "search", "kind": "ammo", "name": "web_search",
      "config": { "inputSchema": {"...": "..."}, "code": "..." } }
  ],
  "edges": [
    { "id": "e1", "type": "delegate", "from": "king", "to": "research",
      "config": { "task": "Research {{company}}" } },
    { "id": "e2", "type": "check", "from": "research", "to": "factcheck",
      "config": { "maxRetries": 2 } }
  ]
}
```

One JSON file. Prompts and code are strings inside it. Exporting the graph
gives you runnable code; importing code back is out of scope for v1.

## 4. Runtime (how a run works)

1. **Input gates.** Every Warden with an `input` gate runs on the user message. `block` stops the run.
2. **King loop.** King reads the prompt, picks an action: call a tool, delegate to a Knight, or finish.
3. **Delegate.** Knight runs its own small loop with only the tools it has `use` edges to.
4. **Check.** If the Knight has a `check` edge, the Advisor sees the output. `feedback` re-runs the Knight with the note appended. Stops at `maxRetries`.
5. **Return.** Knight result goes back to King as a tool-result message.
6. **Output gates.** `output` Wardens run on the final answer. `rewrite` edits it, `block` replaces it.
7. **Done.**

Every step emits an event: `{ts, nodeId, edgeId, phase: start|token|end|error, input, output, tokens, ms}`.
The UI subscribes to this stream. That is the whole observability story.

## 5. UI flow

| Step | Screen | What the user does |
|---|---|---|
| 1 Brief | Prompt box | Say what to build |
| 2 Plan | Plan card | Read the proposed agents, tools, checks. Accept or edit. |
| 3 Graph | Canvas | See the generated graph. Add / remove nodes and edges. |
| 4 Edit | Canvas + inspector | Click a node or edge, edit prompt / schema / code inline |
| 5 Playground | Input + canvas + response | Paste API key, run sample input, watch nodes light up while the answer streams |
| 6 Inspect | Node output panel | Click any node, see every call it made: input, output, tokens, time |

Steps 3–6 share the same canvas. The step only changes the side panels.

## 6. Architecture (v1)

- **Frontend:** single-page app. Graph state in one store. Canvas in SVG.
- **Runner:** small server (Node or Python). Takes the graph JSON + input + API key.
  Runs the loop from section 4. Streams events over SSE.
- **Storage:** graph JSON in local files first. Database later.
- **Planner:** one LLM call with a fixed output schema that emits a graph JSON. The plan
  in step 2 is a text rendering of that JSON.

## 7. Critique (honest)

**Strong points**
- Typed edges make behaviour explicit. Most agent frameworks hide the loop in code.
- Per-node observability falls out of the event stream for free.
- Non-engineers can change a prompt without touching code.

**Weak points / risks**
1. **Metaphor tax.** King / Knight / Warden is fun but users must learn two vocabularies
   (yours and the real one). Show the real term next to the metaphor, or drop the metaphor for v2.
2. **Graph ≠ program.** Loops, conditionals, and parallel fan-out are hard to draw and harder
   to read. The `check` loop already needs a retry cap. Real systems need branching on
   tool results. Decide early: is the graph *the* program, or a view of a program?
   Recommendation: the graph is the program, but only for a small fixed set of edge
   types. Anything else is code inside a node.
3. **Code in a text box.** Tools and validators are real code. A textarea is not an IDE.
   You will need syntax highlighting, tests, and a sandbox to run untrusted code safely.
4. **Static graphs lie at runtime.** A King with 5 Knights may only use 2 on a given run.
   The "lit up" view is honest; the static view is not. Show run coverage on the canvas.
5. **Council (Advisor) and Court (Warden) overlap.** Both are "check something and act."
   Difference in v1: Advisors loop back to the producer with feedback; Wardens gate
   input/output with block / rewrite. Keep that rule strict or merge them.
6. **API keys in the browser.** Fine for a playground. Not fine for a product.
   Route through the runner and never log the key.
7. **"Plan → graph" will be wrong sometimes.** The planner LLM will emit graphs that are
   too big or miss a guardrail. Validate the JSON against a schema and lint it (one King,
   no orphan nodes, every Knight has at least one edge in).
8. **Scope creep is the main risk.** v1 should be: one King, N Knights, tools, one Advisor
   type, one Warden type, one model provider. Ship that, then grow.

## 8. What the mock shows

`index.html` is a runnable, dependency-free mock of all six steps with a sample
"company memo" graph and a scripted run. Nothing calls a real API yet.
