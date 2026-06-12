# Session 8 — Assignment queries (theme: world cities / travel briefings)

Run every query from `code/` with the gateway already up
(`cd gateway && uv run main.py` in another terminal):

```bash
cd code
uv run python flow.py "<query>"
```

Each run prints its node lines and a FINAL block, and persists the full
trace under `code/state/sessions/<sid>/`. Walk any trace with
`uv run python replay.py <sid>`.

---

## Part 1 — Five base queries (verbatim, within bounds)

| id | query | expected shape | bound |
|----|-------|----------------|-------|
| hello | `Say hello.` | planner → formatter (2 nodes) | < 3 s |
| A | `Fetch https://en.wikipedia.org/wiki/Claude_Shannon and tell me his birth date, death date, and three key contributions to information theory.` | planner → researcher → distiller → (critic) → formatter | S7 carryover |
| I | `Find the current populations of Tokyo, Delhi, and Cairo and tell me which is largest.` | planner → 3× researcher (parallel) → coder → formatter+sandbox | parallel layer = max(branches) |
| J | `Read /nonexistent/path.txt and tell me what's in it.` | planner → formatter (degenerate DAG, no tool dispatched) | 2 nodes |
| K | `For Lagos, Cairo, and Kinshasa, find current populations and growth rates and tell me which is growing fastest.` | parallel researchers → coder → formatter; SIGKILL mid-run, then `--resume` | resume re-runs only the killed node |

K resume procedure:
```bash
uv run python flow.py "For Lagos, Cairo, and Kinshasa, find current populations and growth rates and tell me which is growing fastest." &
# kill the process mid parallel-layer, note the <sid> printed at the top
uv run python flow.py --resume <sid>
```

---

## Part 2 — Parallel fan-out (≥ 3 independent concurrent nodes)

```
Build a travel snapshot: find the current population of Tokyo, the elevation of La Paz, and the year Cairo was founded.
```

Three genuinely independent researcher sub-tasks (no data dependency
between them) → one formatter. **Proof to capture from the trace:** the
three researcher nodes all enter `running` in the *same* executor batch,
and the parallel layer's wall-clock ≈ the slowest single branch, not the
sum of the three. Read the per-node `started_at` / `completed_at` in
`state/sessions/<sid>/nodes/` to show the overlap.

---

## Part 3 — Critic verdict (one pass, one fail + recovery)

Property the Critic can verify **by reading** (structural completeness /
support against source — the cases the codebase documents as reliable):

```
Summarise Kyoto as a record with exactly these fields: country, population, founded_year, sister_cities. All four fields are required.
```

The Distiller is `critic:true`, so the orchestrator auto-inserts a Critic
between Distiller and Formatter.

- **PASS run:** upstream findings contain all four fields → Distiller
  fills them → Critic reads, every field supported → `pass` → Formatter.
- **FAIL run:** when a required field is unsupported/missing in the
  Distiller output, the Critic emits `fail` → orchestrator splices a
  recovery Planner (capped at one re-plan/branch) → corrected answer.

Capture both traces. The fail trace must show the `critic-fail recovery:
planner node …` line and a corrected final answer.

> Note: the Critic is the generic read-verifier. We deliberately chose a
> *structural-completeness* property (required keys present + supported),
> which the model can evaluate by reading — not arithmetic/syllables,
> which the codebase honestly flags as a Critic blind spot.

---

## Part 4 — Coder (computation the Formatter can't do from text)

```
Here are three city populations — Tokyo 37,400,068; Delhi 32,900,000; Cairo 22,100,000. Compute the mean, the population standard deviation, and which city is furthest from the mean.
```

Planner emits a `coder`; the orchestrator auto-runs `sandbox_executor`
on its Python. The program embeds the three numbers as literals, computes
`statistics.mean` / `pstdev` and the argmax of `abs(pop-mean)`, and prints
one JSON line the Formatter lifts. Exact stdev over a list is precisely
what an LLM Formatter cannot produce reliably from prose.

---

## Part 5 — New skill: translator

```
Translate the traveller's phrase "Where is the train station?" into the local languages of Tokyo, Paris, and Cairo.
```

Planner resolves the cities to Japanese / French / Arabic and emits a
`translator` node scoped by `metadata.question`; the Formatter renders
the three translations. **No orchestrator code was modified** — only a
yaml entry (`agent_config.yaml`), a prompt file (`prompts/translator.md`),
the Planner's skill list (a prompt), and a gateway routing pin.
