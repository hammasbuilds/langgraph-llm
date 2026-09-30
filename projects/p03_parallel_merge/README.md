# 03 · The synthesis that reads as complete when a branch failed (LangGraph, LangChain, Ollama, FastAPI)

**Every single silent branch failure produced a synthesis report covering all four aspects by
name, with zero indication anything had gone wrong. Explicitly instructing the synthesis to
check for gaps caught it half the time.**

Four analyst nodes run as genuine parallel branches in a LangGraph fan-out, each covering one
aspect of a company (financial, technical, market, team). A synthesis node reads all four and
writes one report. One branch is made to fail — standing in for a timed-out tool call or an
errored API — and the question is whether the report says so.

---

## Results

| strategy | mean aspects represented (of 4) | disclosed the failure | reads as complete |
|---|---|---|---|
| `all_succeed` | 4.00 | n/a | n/a |
| `fail_silent` | 3.00 | **0%** | **4/4** |
| `fail_flagged` | 3.00 | 50% | 2/4 |

`all_succeed` is the control: every branch worked, every report mentioned all four aspects.
`fail_silent` runs the identical graph with one branch producing nothing and the synthesis
prompt unchanged. In **all four** briefs tested, the report read as a complete four-aspect memo
— financial, technical, market and team all mentioned by name — despite one of those aspects
having received literally nothing to synthesise from.

```
fail_silent  b1  financial branch failed  ->  report covers technical, market, team.
                                               Financial section: not mentioned as missing.
fail_silent  b2  technical branch failed  ->  report covers financial, market, team.
                                               Technical section: not mentioned as missing.
```

Nothing in the graph crashed. Nothing in the output looks wrong unless you already know which
branch to check.

## Telling it to check for gaps works — half the time

`fail_flagged` adds one instruction to the synthesis prompt: *if any section is empty or
missing, say so explicitly and name it.* That took disclosure from 0% to 50%:

| brief | failed branch | disclosed under `fail_flagged`? |
|---|---|---|
| b1 | financial | no |
| b2 | technical | **yes** |
| b3 | market | no |
| b4 | team | **yes** |

There is no obvious pattern in which two it caught — not the first two, not any particular
aspect. An instruction to check for gaps is not a mechanism that reliably finds them; it raises
the rate from never to sometimes, on identical inputs to the failure it is meant to catch.

## Where this shows up in LangGraph

The graph itself does the right thing mechanically. Four analyst nodes run in the same
superstep — genuine parallelism, confirmed by state updates arriving from all four before the
synthesis node runs at all — and a failed branch does not crash the graph or block the others.
That correctness is exactly what makes the failure invisible: the graph completes cleanly,
produces a full state object, and hands the synthesis node a `findings` dict with one empty
value sitting among three full ones. Nothing about the *graph's* execution signals a problem.
Whether that empty value gets *noticed* is now entirely a prompting question, and the default
answer on this run was no, always.

## Run it

```bash
make install
ollama pull qwen2.5:3b-instruct

uv run python -m projects.p03_parallel_merge.benchmark
uv run uvicorn projects.p03_parallel_merge.web:app --port 8113
```

The UI runs the fan-out live and shows all four branches' findings beside the synthesised
report, so a failed branch's empty box sits right next to a report that never mentions it:

![a failed branch, and a synthesis report that reads as fully complete anyway](../../screenshots/p03-2-silent-failure-light.png)

## Scope

- **Four briefs, one failed branch each.** Enough to show 4/4 silent and 2/4 flagged; not
  enough to put a precise rate on either. The direction — silent gating is unreliable, explicit
  flagging is better but not sufficient — is the claim, not the exact percentages.
- **The failure is simulated as an empty string**, not a raised exception. A LangGraph node
  that actually raises stops the whole run rather than reaching synthesis; the case tested here
  is the more dangerous one, where a wrapped tool call swallows its own error and returns
  nothing, and the graph never learns anything went wrong at all.
- **One model.** Whether a larger model's synthesis reliably notices a missing section is
  worth testing directly rather than assuming from this run.
- **It does not test more than one simultaneous failure**, or a partial failure (a branch that
  returns some but not all of what it should).
