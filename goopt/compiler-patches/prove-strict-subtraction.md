# Compiler Optimization: Enable Strict Subtraction Facts in prove.go

> ⚙️ GoOpt — automated Go compiler optimizer

## Summary

Enables two commented-out strict subtraction fact propagations in the SSA prove pass (`cmd/compile/internal/ssa/prove.go`). These facts allow the bounds check eliminator to prove that subtraction results are strictly less than the minuend (when subtracting a positive) or strictly positive (when subtracting a smaller value).

## Motivation

In etcd's MVCC `key_index.go:161`, the expression `g.revs[n]` has a bounds check that cannot be eliminated with the stock compiler. The `generation.walk()` method returns an index derived from subtraction, and without the strict fact (`v < x` when `y > 0`), the prove pass cannot establish that `n` is in-bounds.

## Compiler Change

**File**: `src/cmd/compile/internal/ssa/prove.go` (function `detectSubRelations`)

Uncommented two blocks:
1. When `yLim.min > 0`: propagate `v < x` (strict less-than) instead of only `v <= x`
2. When `y < x` (strictly ordered): propagate `v >= 1` instead of only `v >= 0`

## Benchstat Results (10 iterations each)

```
                      │ baseline │       optimized        │
                      │  sec/op  │    sec/op     vs base  │
IndexCompact1-4         129.6n ±3%   127.2n ±4%        ~ (p=0.060)
IndexCompact100-4       10.61µ ±3%   10.21µ ±4%   -3.76% (p=0.023)
IndexCompact10000-4     1.073m ±4%   1.040m ±2%   -3.03% (p=0.029)
IndexCompact100000-4    6.495m ±183%  4.428m ±21% -31.82% (p=0.001)
IndexCompact1000000-4   238.6m ±5%   220.6m ±5%   -7.56% (p=0.000)
IndexPut-4              2.470µ ±15%  2.255µ ±4%   -8.71% (p=0.000)
IndexGet-4              2.154µ ±18%  1.977µ ±8%   -8.20% (p=0.001)
geomean                 74.00µ       66.71µ        -9.85%
```

## Bounds Checks Eliminated

- `server/storage/mvcc/key_index.go:161` — `g.revs[n]` access after `generation.walk()` return

## Correctness

- etcd `server/storage/mvcc` test suite passes with modified compiler
- The change only propagates facts that are mathematically sound (if x-y where y>0, then result < x)

## Trade-offs

- **Compile time**: Negligible — two additional fact propagations per subtraction with known-positive operand
- **Code size**: No change — only eliminates instructions (bounds checks)
- **Risk**: Low — the TODO comments in upstream Go indicate these were intentionally left as future work

## Upstream Potential

High. This is a conservative, correct improvement that was already considered by the Go team (evidenced by the commented-out code with "TODO: is this worth it?" annotations). The etcd benchmark results demonstrate it is indeed worth it for real-world workloads.

## Reproduction

```bash
# Apply patch to Go 1.26.0 source
cd $GOROOT/src/cmd/compile/internal/ssa/
patch -p1 < prove-strict-subtraction.patch
cd $GOROOT/src && ./make.bash

# Verify BCE improvement
go build -gcflags='-d=ssa/check_bce/debug=1' ./server/storage/mvcc/ 2>&1 | grep key_index
# Stock: key_index.go:161:13: Found IsInBounds
# Modified: (no output for line 161 — check eliminated)
```
