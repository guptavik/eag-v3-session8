# EAGV3 Session 8 — Multi-Agent DAG Orchestration

A growing-graph agent: the **Planner** emits a DAG of typed skill nodes,
the **Executor** runs every node whose predecessors are complete
(parallel siblings via `asyncio.gather`), a **Critic** sits between a
flagged producer and its successor, and the whole graph is persisted to
disk so a killed run resumes. Theme for the worked queries: **world
cities / travel briefings**.

All raw traces referenced below live in [`logs/`](logs/) (stdout `.log`
+ stderr `.err`, split) and under `code/state/sessions/<sid>/` (graph +
per-node JSON, fully reconstructible offline).

---

## What I built (the deliverables)

| Part | Deliverable | Files touched |
|------|-------------|---------------|
| 4 | **Coder skill** — real prompt emitting stdlib Python for the sandbox | [`code/prompts/coder.md`](code/prompts/coder.md) |
| 5 | **New skill: `translator`** — yaml entry + prompt + planner awareness | [`code/agent_config.yaml`](code/agent_config.yaml), [`code/prompts/translator.md`](code/prompts/translator.md), [`gateway/agent_routing.yaml`](gateway/agent_routing.yaml) |
| 3 | Planner-prompt fixes so a Critic gate wires correctly | [`code/prompts/planner.md`](code/prompts/planner.md) |

**The orchestrator was never modified.** `git status` shows changes only
in yaml + prompt files (and runtime artifacts `state/*`, `usage.json`).
No edits to `flow.py`, `skills.py`, `recovery.py`, `persistence.py`,
`sandbox.py`, `mcp_runner.py`. The recovery classifier's **22 unit tests
still pass** (`uv run python -m pytest tests/test_recovery.py -q`).

---

## How to run

```bash
# Terminal 1 — gateway (reads ../.env automatically)
cd gateway && uv run main.py            # boots on :8108

# Terminal 2 — agent. On Windows use UTF-8 mode for the box-drawing output:
cd code
PYTHONUTF8=1 PYTHONIOENCODING=utf-8 uv run python flow.py "Say hello."
uv run python replay.py <sid>           # walk any trace
```

Environment used: Python 3.11 (uv venv), Ollama `nomic-embed-text` for
embeddings, gateway provider ladder `cerebras, groq, nvidia, github`
(+ Tavily for web search). The exact queries are in [`QUERIES.md`](QUERIES.md).

---

## Part 1 — Five base queries (verbatim, within bounds)

| id | query | DAG shape | nodes | result | log | sid |
|----|-------|-----------|-------|--------|-----|-----|
| **hello** | `Say hello.` | planner → formatter | 2 | `Hello!` | [hello.log](logs/hello.log) | s8-4ddacaef |
| **A** | `Fetch …/Claude_Shannon and tell me his birth date, death date, and three key contributions…` | planner → researcher → distiller → formatter | 4 | born 1916-04-30, died 2001-02-24, + 3 contributions (Math. Theory of Communication 1948, entropy, channel capacity) | [A_shannon.log](logs/A_shannon.log) | s8-6ee0119f |
| **I** | `Find the current populations of Tokyo, Delhi, and Cairo and tell me which is largest.` | planner → 3× researcher (parallel) → formatter | 5 | Tokyo largest (~37 M) | [I_populations.log](logs/I_populations.log) | s8-7a97e92b |
| **J** | `Read /nonexistent/path.txt and tell me what's in it.` | planner → formatter (degenerate, no tool dispatched) | 2 | explains path can't be accessed | [J_graceful.log](logs/J_graceful.log) | s8-0bbb04cf |
| **K** | `For Lagos, Cairo, and Kinshasa, find current populations and growth rates and tell me which is growing fastest.` | parallel researchers → coder → formatter (+sandbox); SIGKILL + resume | 7 | Kinshasa fastest | [K_resume.log](logs/K_resume.log) | s8-880b4c12 |

**hello** is the minimum DAG; **J** is graceful fail-fast (the Planner
sees an unanswerable request and emits a Formatter directly — *no tool
dispatched*). **A** regresses cleanly through the DAG with named nodes.

### Query K — resumable execution (SIGKILL + `--resume`)

The first process was killed with a hard `Stop-Process -Force` while the
three researchers were mid-flight. The graph on disk at the moment of
kill (the durable artifact):

```
n:1 planner    complete
n:2 researcher running     ← interrupted
n:3 researcher running     ← interrupted
n:4 researcher running     ← interrupted
n:5 coder      pending
n:6 formatter  pending
```

`flow.py --resume s8-880b4c12` reset the three `running` researchers to
`pending`, re-ran them, then ran coder → formatter → sandbox_executor.
Final answer correctly names **Kinshasa** as fastest-growing. The entire
run is reconstructible from `state/sessions/s8-880b4c12/` without the
process. (Resume re-runs killed researchers from the top — mid-tool-call
resume is a documented deferral.)

---

## Part 2 — Parallel fan-out (wall-clock = max of branches, not sum)

Query (three **independent, heterogeneous** sub-tasks):

> `Build a travel snapshot: find the current population of Tokyo, the elevation of La Paz, and the year Cairo was founded.`

The Planner emitted **three concurrent researcher nodes** (`metadata.question`
scoped, no `USER_QUERY` leakage) → one formatter. Answer: Tokyo ≈14.1 M
(metro 37 M), La Paz ≈3,650 m, Cairo founded 969 CE.
Log: [P2_fanout.log](logs/P2_fanout.log) · sid `s8-f64e3750` · timing [P2_timing.txt](logs/P2_timing.txt)

```
node   skill        start     end    dur
n:1    planner        0.0     4.5    4.5
n:2    researcher    12.8    45.6   32.8   ┐
n:3    researcher     4.7    45.6   40.9   ├ all three in flight together,
n:4    researcher    17.2    45.6   28.5   ┘ all finish at t=45.6
n:5    formatter     45.6    49.0    3.4

researcher layer: wall-clock span = 40.9s  ==  max branch (40.9s)
                   sum-of-branches = 102.2s  →  parallel saved 61.3s
```

The parallel layer's wall-clock equals the **slowest branch (40.9 s)**,
not the **sum (102.2 s)**. Base query I shows the same property (span
37.0 s vs sum 90.7 s — [I_timing.txt](logs/I_timing.txt)).

---

## Part 3 — Critic verdict (one pass, one fail + recovery; same query)

Property the Critic verifies **by reading** — structural completeness:
"the record must contain all required fields, each supported by the
source." (This is the Critic's reliable zone; arithmetic/syllable
properties are its documented blind spot, deliberately avoided.)

> `Summarise Lyon, France as a record with exactly these required fields: country, population, founded_year, and sister_cities. All four fields are required and must be supported by the source.`

**PASS run** — [P3_pass.log](logs/P3_pass.log) · sid `s8-95fc1978`:

```
planner → researcher → distiller → critic → formatter
critic verdict = pass  ("all four required fields present and supported")
formatter.inputs = [USER_QUERY, n:3 distiller (DATA), n:4 critic (GATE)]
→ correct record (France / 519,127 / 43 BC / sister cities)
```

**FAIL → recovery → corrected answer** — same query, [P3_fail.log](logs/P3_fail.log) · sid `s8-771333eb`:

```
n:7  researcher  → n:8  critic  FAIL  ("findings '(not found)' — no fields supported")
        ↪ critic-fail recovery: planner node n:10 spliced
n:11 researcher  → n:12 critic  FAIL  ("narrative findings, no structured record")
        ↪ critic-fail recovery: planner node n:14 spliced
n:15 researcher  → n:16 distiller → n:17 critic  PASS
n:18 formatter (reads distiller n:16 + critic n:17) → CORRECTED answer
```

The fail genuinely splices a Planner recovery (capped at one re-plan per
branch), and the recovery's added **distiller** converts researcher prose
into the structured record the Critic accepts → corrected final answer.

> Planner-prompt fix behind this: a Critic emits only `{verdict,
> rationale}` and carries no data, so the Formatter must take its DATA
> from the **producer** (distiller) and only its **gate** from the
> Critic — `formatter.inputs = [USER_QUERY, n:<producer>, n:<critic>]`.
> Before the fix the Planner wired the Formatter to read the Critic and
> the answer came back "no data".

---

## Part 4 — Coder: computation the Formatter can't do from text

> `Given these three city populations — Tokyo 37,400,068; Delhi 32,900,000; Cairo 22,100,000 — compute the exact mean, the population standard deviation, and which city is furthest from the mean.`

The Coder emits stdlib Python (`statistics`), the SandboxExecutor runs it
and prints one JSON line, the Formatter lifts it. This is a clean
**before/after** that proves the assignment's premise
([P4_before_after.txt](logs/P4_before_after.txt)):

| run | formatter reads | reported σ | correct? |
|-----|-----------------|-----------|----------|
| `s8-fb5b67f3` (before) | the **coder** (source code) | 6,432,040 | ❌ formatter recomputed by hand, **wrong** |
| `s8-e91fabb8` (after) | the **sandbox** (executed result) | **6,420,303.67** | ✅ matches sandbox stdout |

Final (correct) run — [P4_coder.log](logs/P4_coder.log) · sid `s8-e91fabb8`:

```
planner → coder(n:2) → sandbox_executor(n:3) → formatter(n:4 reads n:3)
sandbox stdout: {"mean":30800022.67,"population_stdev":6420303.67,
                 "furthest_from_mean":{"city":"Cairo","distance":8700022.67}}
FINAL: mean 30,800,022.67 · σ 6,420,303.67 · Cairo furthest
```

The before-run shows the Formatter getting the standard deviation
**wrong** when it only sees code — exactly the failure the Coder exists
to prevent.

---

## Part 5 — New skill: `translator` (no orchestrator change)

Added entirely as a **yaml entry + prompt file** (+ a one-line gateway
pin and the Planner's skill list). The catalogue had research / distill /
summarise / critic / format / code — nothing for translation.

> `Translate the traveller's phrase "Where is the train station?" into the local languages of Tokyo, Paris, and Cairo.`

Log: [P5_translator.log](logs/P5_translator.log) · sid `s8-435f793d`. The
Planner resolved each city to its language and fanned out **three parallel
translator nodes** → formatter:

```
n:2 translator (Japanese): 駅はどこですか？
n:3 translator (French):   Où est la gare ?
n:4 translator (Arabic):   أين محطة القطار؟
```

Adding the skill required **zero** changes to the Executor — confirming
the "skills are yaml + prompts" contract.

---

## Architectural contract — verification

- **Orchestrator untouched.** `git status` lists only `agent_config.yaml`,
  `prompts/*.md`, `gateway/agent_routing.yaml` (+ runtime `state/`,
  `usage.json`). No `flow.py` / `skills.py` / `recovery.py` /
  `persistence.py` / `sandbox.py` edits.
- **Recovery classifier intact.** `pytest tests/test_recovery.py` → **22
  passed**, after all prompt changes.
- **Planner emits the graph; Executor runs it; Critic sits between a
  flagged producer and its successor** — all demonstrated above.

---

## Reportable findings (gateway-side; not patched per the brief)

1. **`nvidia` provider crashes the tool-use loop.** Any tool-using skill
   (retriever, researcher) that fails over to `nvidia` raises
   `ExceptionGroup: unhandled errors in a TaskGroup`; `groq`, `gemini`,
   `cerebras` all handle it fine (reproduced directly per-provider). In
   the Part-3 fail trace the retriever hit this and the **orchestrator
   recovered gracefully** (upstream_failure → replan), so it's resilient,
   but the provider should fail over instead of raising. `retriever` was
   re-pinned `github → groq` (config only) to reduce the hit rate.

2. **Double `sandbox_executor` on the explicit compute chain.** Because
   `coder` has a static `internal_successors: [sandbox_executor]`, when
   the Planner *also* emits an explicit sandbox node (needed so the
   Formatter can reference the executed result), the code runs twice — one
   feeds the Formatter, one is a redundant leaf (idempotent, ~0 s). The
   root cause is that the auto-appended sandbox node is unnameable by the
   Planner, so the executed result can't otherwise reach the Formatter. A
   cleaner design would let the Planner reference the auto-appended node,
   or skip the internal successor when the Planner wires its own.

3. **Windows:** the orchestrator prints box-drawing characters; run with
   `PYTHONUTF8=1` (cp1252 console otherwise errors). Harmless asyncio
   proactor-transport `ResourceWarning`s print to stderr at interpreter
   shutdown after MCP subprocess use — cosmetic, captured to `.err` files.
