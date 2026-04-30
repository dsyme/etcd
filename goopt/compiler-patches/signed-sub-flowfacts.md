# Signed Subtraction Flow Facts in prove.go

> ⚙️ GoOpt — automated Go compiler optimizer

## Summary

Add signed flow facts for subtraction operations in the SSA prove pass. When both operands of a subtraction are known to be non-negative, the compiler can now derive that:
1. The result is <= the left operand (always safe since no overflow is possible)
2. If the left operand's minimum >= right operand's maximum, the result is non-negative

## Motivation

etcd's MVCC hot paths (particularly `(*keyIndex).compact` and `(*generation).walk`) perform extensive index arithmetic with patterns like `l-i-1` where both `l` and `i` are non-negative loop variables. The prove pass previously only derived unsigned facts for subtractions but not signed facts, limiting bounds check elimination for signed index expressions.

## Compiler File Modified

- `src/cmd/compile/internal/ssa/prove.go` (function `addLocalFacts`, OpSub case at line ~2511)

## Benchstat Results

```
goos: linux
goarch: amd64
pkg: go.etcd.io/etcd/server/v3/storage/mvcc
cpu: AMD EPYC 7763 64-Core Processor
                      │  baseline  │          optimized           │
                      │   sec/op   │   sec/op     vs base         │
IndexCompact1-4          130.7n ±2%   134.8n ± 5%       ~ (p=0.167)
IndexCompact100-4        10.82µ ±1%   10.76µ ± 2%       ~ (p=0.645)
IndexCompact10000-4      1.092m ±3%   1.073m ± 6%  -1.75% (p=0.015)
IndexCompact100000-4     4.346m ±27%  3.761m ± 4% -13.48% (p=0.050)
IndexCompact1000000-4    175.8m ±10%  159.6m ± 4%  -9.25% (p=0.010)
IndexPut-4               2.104µ ±7%   1.995µ ± 2%  -5.18% (p=0.038)
IndexGet-4               1.776µ ±8%   1.677µ ± 2%  -5.57% (p=0.027)
geomean                  64.03µ       60.95µ       -4.81%
```

## Correctness

- etcd `server/storage/mvcc` tests pass with modified compiler
- WAL benchmarks show no regression
- The optimization is sound: for x >= 0, y >= 0, x - y <= x always holds (no overflow possible since max result is x which fits in the signed type)

## Trade-offs

- **Compile time**: Negligible — adds two conditional fact updates per subtraction
- **Code size**: No change (only affects prove pass decisions, not code generation directly)
- **Risk**: Low — the derived facts are mathematically correct for non-negative operands

## Upstream Potential

High. This addresses a documented FIXME in the Go compiler source and provides general benefit for any code performing arithmetic on non-negative integers (array indexing, length calculations, etc.).
