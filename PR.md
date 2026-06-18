# PR title

Elaboration: make ENABLED-axioms usage detection linear in the context size

---

# PR body

## A note up front

I'm a *user* of TLAPS, not a contributor — I don't know the prover's
implementation. I hit a real, reproducible slowdown on one of my specs, used
[Claude Code](https://claude.com/claude-code) to profile `tlapm`, locate the
root cause, and write the fix below, and I reviewed and validated the result
myself (build + full test suite, see *Testing*).

I'm aware the AI-assisted-contribution policy on the `tlaplus` repos isn't
settled yet. I'm opening this anyway because (a) I genuinely need the fix and
(b) it's small and self-contained (1 file, ~23 lines, no behaviour change), so
it should be cheap to review. I'm very happy to discuss, rework, or split it,
and I hope an AI-generated patch isn't taken the wrong way — the intent is just
to surface a concrete, isolated performance bug with a minimal fix attached.

## Summary

`check_usable` in the ENABLED-axioms-usage detector
(`Module.Elab.check_enabled_axioms_usage`, `src/module/m_elab.ml`) runs for
**every `BY`/`OBVIOUS`** and, for **each fact** in the proof context,
recomputes `cx_front cx (size - n)` — an O(context) slice — to resolve the
fact's De Bruijn reference. That makes the scan **O(context × #facts)** per
proof. This patch rewrites it as **two linear passes** over the context, which
is **semantically identical** but O(context).

## The spec class that triggers it

This shows up on **hierarchical refinement specs that carry several refinement
properties in a single module** — i.e. a module with multiple `INSTANCE`
statements, each importing a whole sub-module hierarchy (with its theorems).
Each `INSTANCE` brings the instantiated modules' theorems into **every**
obligation's context as **hundreds of facts**, so the per-fact O(context) work
in `check_usable` explodes. A flat spec, or one with few/no instances, never
feels it.

## Root cause

For each fact `Fact (Ix k, …)` at context front-index `fi`, the original code
did roughly:

```ocaml
let cx_ = Expr.T.cx_front cx ((Deque.size cx) - n) in   (* O(context) *)
get_val_from_id cx_ n                                    (* check Bpragma "ENABLEDaxioms" *)
```

inside a `Deque.iter` over the whole context. So resolving N facts against a
context of size N is O(N²), and with instance-imported theorems N is in the
hundreds for every obligation.

## The fix

Two linear passes (O(context)):

1. mark the context positions that hold an `ENABLEDaxioms` pragma
   (`Defn (Bpragma ("ENABLEDaxioms", …), …)`), then
2. for each `Fact (Ix k, …)` at front-index `fi`, check whether its target
   front-index `fi - k` is one of the marked positions.

The resolved target (`fi - k`) is exactly what
`get_val_from_id (cx_front cx (size - fi)) k` computed before — this is a pure
algorithmic rewrite, not a behaviour change.

## Performance

Measured with `--timing` on one obligation of the spec, instances active
(Z3 backend; the obligation itself solves in ~4ms, so this is all `tlapm`-side
elaboration):

| phase | before | after |
| --- | --- | --- |
| `check_usable` (ENABLED scan) | 25.1 s | 0.11 s |
| `analysis` (whole elaboration phase) | 31 s | 5.3 s (~6×) |
| single-obligation check, total | 38.2 s | 15.5 s (~2.5×) |

Elaboration is paid on **every** `tlapm` invocation — even a `--toolbox`
single-obligation check re-elaborates the whole module — so this directly
speeds up interactive proof development on instance-heavy specs, not just batch
runs.

## Testing

All green, before and after:

- `dune runtest` — unit tests (145).
- `cd test && env TEST_CASE=./soundness_tests dune runtest -f` — 281 cases (14 skipped).
- `env TEST_CASE=./fast/enabled_cdot dune runtest -f` — **14/14**, including the
  cases that exercise the ENABLED path (`ExpandOnlyENABLED`, `NestedENABLED`,
  `NestedENABLED_from_AutoUSE`, `Level_of_parametric_INSTANCE`).
- A spec with no instances (Euclid) is unaffected, as expected.

## Two related hot spots (same pattern, left untouched)

While profiling I found the same per-fact `cx_front`/`scx_front` shape in two
other places. Neither was hot for my spec, so I left them out to keep this PR
minimal, but flagging them in case they matter for ENABLED-heavy specs:

1. **`check_enabled_axioms_map#check_usable`** (`src/module/m_elab.ml`,
   ~lines 851 / 904 / 917) — the sibling visitor, with an identical
   O(context × #facts) scan. It only runs when ENABLED axioms are actually
   *used*, so it didn't fire on my spec.
2. **Level computation** (`src/expr/e_levels.ml`, `method expr`, ~lines 297 / 311)
   — `scx_front`/`cx_front` recomputed per De Bruijn reference, i.e.
   O(refs × context).

Happy to follow up on those separately if it's useful.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
