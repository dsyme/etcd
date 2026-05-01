# Leaf Function Inline Bonus

> ⚙️ GoOpt — automated Go compiler optimization

## Summary

Add an extra inlining budget of 8 units for "leaf" functions — functions that contain no non-cheap function calls. This allows small, compute-only functions that slightly exceed the default budget (80) to be inlined, improving performance of hot paths.

## Motivation

In etcd's MVCC index, `(*keyIndex).findGeneration` is called on every key lookup (`get`, `compact`). It has an inlining cost of 81 — just 1 unit over the default budget of 80. The function is a pure loop over slice data with no calls, making it an ideal inlining candidate. Inlining eliminates call overhead and enables further optimizations (register allocation across the call boundary, bounds check elimination with caller context).

## Compiler Files Modified

- `src/cmd/compile/internal/inline/inl.go`: Added `hasCall` field to `hairyVisitor`, `inlineLeafBonus` constant (8), leaf bonus check in `tooHairy()`, and adjusted `inlineCostOK()` to allow leaf functions up to cost 88.
- `src/cmd/compile/internal/ir/func.go`: Added `IsLeaf` field to `Inline` struct.

## Benchstat Results

```
goos: linux
goarch: amd64
pkg: go.etcd.io/etcd/server/v3/storage/mvcc
cpu: AMD EPYC 9V74 80-Core Processor
                      │  baseline   │          optimized           │
                      │   sec/op    │    sec/op     vs base        │
IndexCompact1-4          128.7n ± 2%   125.8n ± 2%  -2.25% (p=0.008 n=10)
IndexCompact100-4        10.26µ ± 2%   10.27µ ± 2%       ~ (p=0.971 n=10)
IndexCompact10000-4      872.9µ ± 7%   884.0µ ± 1%       ~ (p=0.105 n=10)
IndexCompact100000-4     5.902m ± 17%  6.310m ± 16%      ~ (p=0.075 n=10)
IndexCompact1000000-4    232.2m ± 3%   214.9m ± 4%  -7.48% (p=0.000 n=10)
IndexPut-4               2.114µ ± 4%   2.050µ ± 4%  -3.00% (p=0.014 n=10)
IndexGet-4               1.809µ ± 3%   1.783µ ± 3%       ~ (p=0.138 n=10)
geomean                  66.97µ        66.35µ        -0.94%
```

## Key Improvements

- **IndexCompact1000000**: -7.48% (p=0.000) — large compaction workload
- **IndexPut**: -3.00% (p=0.014) — key insertion
- **IndexCompact1**: -2.25% (p=0.008) — single key compaction

## Correctness

- All `server/storage/mvcc` tests pass with modified compiler
- No regressions in other benchmarks

## Trade-offs

- **Code size**: Minimal increase — only affects functions with cost 81-88 that are true leaves
- **Compile time**: Negligible — leaf detection adds one boolean check per function
- **Scope**: Conservative — only 8 units of extra budget, only for call-free functions

## Upstream Potential

**High**. This is a targeted, well-motivated heuristic improvement to the Go inliner. Leaf functions are inherently safe to inline (no call graph expansion risk) and the bonus is small enough to avoid code bloat. The pattern of "useful function barely exceeds budget" is common in real-world code.
