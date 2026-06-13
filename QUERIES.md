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
Summarise Lyon, France as a record with exactly these required fields: country, population, founded_year, and sister_cities. All four fields are required and must be supported by the source.
```

The Planner emits a `critic` reading the producer (distiller) and wires
the formatter to depend on BOTH the producer (data) and the critic
(gate): `formatter.inputs = [USER_QUERY, n:<producer>, n:<critic>]`. This
is the planner-emitted critic path (one of the two paths pinned by
`tests/test_recovery.py`; the other is the distiller auto-insert).

- **PASS run:** distiller fills all four fields → Critic reads, every
  field supported → `pass` → Formatter quotes the record.
- **FAIL run (same query):** when a required field is unsupported/missing
  → Critic emits `fail` → orchestrator marks the formatter `skipped` and
  splices a recovery Planner (capped at one re-plan/branch) → the recovery
  adds a distiller path the Critic accepts → corrected answer.

Capture both traces. The fail trace must show the `critic-fail recovery:
planner node …` line and a corrected final answer.

> Note: the Critic is the generic read-verifier. We deliberately chose a
> *structural-completeness* property (required keys present + supported),
> which the model can evaluate by reading — not arithmetic/syllables,
> which the codebase honestly flags as a Critic blind spot.

---

## Part 4 — Coder: the trust-and-verify diamond

```
Find the populations of London, Paris, Berlin and tell me which two are closest in size.
```

The Coder emits `{code, summary}`. The diamond fires: Coder → Formatter
(reads `summary`, the user-facing answer) **and** Coder → SandboxExecutor
(auto-attached `internal_successor`, runs `code` to verify) — both
concurrent. The Formatter quotes the computed difference; the sandbox
independently prints the same integer. Precise multi-digit subtraction is
what an LLM Formatter cannot do reliably from prose; the coder grounds it
in real execution.

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
