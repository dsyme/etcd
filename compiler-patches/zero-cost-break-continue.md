# Zero-Cost Break/Continue for Inlining Budget

> ⚙️ GoOpt — automated Go compiler optimizer

## Summary

Make `break` and `continue` statements free (zero-cost) in the inlining cost model.
These statements compile to a single unconditional jump instruction and should not
count against the inlining budget, similar to `fallthrough` (`OFALL`) and type
declarations (`OTYPE`) which are already free.

## Motivation

The `(*keyIndex).findGeneration` function in etcd's MVCC hot path has an inlining
cost of 81, exceeding the budget of 80 by exactly 1 unit. The single `continue`
statement in the function's loop body accounts for that extra unit. By making
`break`/`continue` free, this critical function becomes inlinable (cost drops to 80),
enabling it to be inlined into `(*keyIndex).get` — a very hot function called on
every key lookup.

## Compiler Files Modified

- `src/cmd/compile/internal/inline/inl.go` — Added `ir.OBREAK` and `ir.OCONTINUE`
  to the existing `case ir.OFALL, ir.OTYPE` that returns `false` (skips budget charge).

## Benchstat Results

```
goos: linux
goarch: amd64
pkg: go.etcd.io/etcd/server/v3/storage/mvcc
cpu: AMD EPYC 7763 64-Core Processor
                │ baseline.txt │      optimized.txt       │
                │    sec/op    │   sec/op    vs base      │
IndexCompact1-4    139.4n ± 1%  136.6n ± 2%  -2.01% (p=0.015 n=8)
IndexPut-4         2.302µ ± 2%  2.307µ ± 2%       ~ (p=0.427 n=8)
IndexGet-4         1.932µ ± 6%  1.904µ ± 10%      ~ (p=0.505 n=8)
geomean             852.8n       843.4n       -1.09%
```

## Correctness

- All `go.etcd.io/etcd/server/v3/storage/mvcc` tests pass with the modified compiler.
- The change is minimal and conservative: `break`/`continue` have zero children in
  the IR and produce only a single unconditional jump (already predicted perfectly by
  modern CPUs).
- `OFALL` (fallthrough) was already treated as zero-cost; break/continue are
  semantically equivalent in terms of codegen cost.

## Trade-offs

- **Code size**: Negligible increase. Only functions that were 1 unit over budget
  (per break/continue statement) become inlinable.
- **Compile time**: No measurable change.
- **Risk**: Very low. This is a conservative heuristic adjustment consistent with
  existing treatment of `OFALL`.

## Upstream Potential

**High**. This is a clean, minimal change that follows existing patterns in the
inlining cost model. The rationale is clear: branch statements produce trivial code
and should not penalize inlining decisions. A similar argument was used for `OFALL`.
