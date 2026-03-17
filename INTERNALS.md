# Show HN: Manifold – a database that acts across connected systems

Manifold is a database that acts.

Stores, retrieves, and executes actions across connected systems deterministically. No LLM at runtime.

Built to solve one problem: agents writing across systems have no ground truth on relationships. They infer at runtime. At scale that's silent state divergence. Manifold puts a verified relationship graph underneath — every write is a graph traversal, not inference.

## Stage 1 — Action factory

Generates typed Python callables from OpenAPI specs or existing modules. Every list, get, update, delete becomes an executable function. Every graph node gets bound to one or more of these at construction time.

## Stage 2 — Per-source graph construction

Discovery runs against each source. List actions called, samples pulled at multiple sizes, field-level statistics computed — cardinality, distribution, null rates, type consistency.

Three-model classifier pipeline:

- Model 1: field naming context — variable names, path structure, conventions
- Model 2: field value statistics — actual data distribution across samples
- Model 3: adjudicates across both, produces confidence-ranked classifications

Output taxonomy: identifier, pseudo-identifier, reference, constant, metric, categorical. Confidence scores applied to build node structure and type relationships.

Numbers: Stripe produces 74 type nodes, 247 field nodes. HubSpot produces 492 nodes total.

Each node bound to its action function. Where no direct update or delete exists, Manifold walks the graph upward to the nearest ancestor that implements one, collects all descendant node values, constructs the payload. Write scope is determined by graph structure, not explicit mapping.

## Classification results

Tested across Stripe, QBO, and HubSpot:
```
┌──────────────┬───────────────┬──────────────┬───────────────┬─────────────────┐
│ Metric       │    Stripe     │     QBO      │    HubSpot    │    Combined     │
├──────────────┼───────────────┼──────────────┼───────────────┼─────────────────┤
│ Strict       │ 52/53 = 98.1% │ 47/47 = 100% │ 40/41 = 97.6% │ 139/141 = 98.6% │
├──────────────┼───────────────┼──────────────┼───────────────┼─────────────────┤
│ Lenient      │ 53/53 = 100%  │ 47/47 = 100% │ 41/41 = 100%  │ 141/141 = 100%  │
├──────────────┼───────────────┼──────────────┼───────────────┼─────────────────┤
│ Ref targets  │ 7/7 = 100%    │ 5/5 = 100%   │ 5/5 = 100%    │ 17/17 = 100%    │
├──────────────┼───────────────┼──────────────┼───────────────┼─────────────────┤
│ Hallucinated │ 0             │ 0            │ 0             │ 0               │
└──────────────┴───────────────┴──────────────┴───────────────┴─────────────────┘
```

98.6% strict accuracy combined. 100% lenient. 100% reference target accuracy across all 17 references. Zero hallucinated classifications across all three sources.

The single strict miss in Stripe and HubSpot respectively are ambiguous fields where naming context and value statistics produce conflicting signals — the exact case the adjudication model exists to handle.

## Stage 3 — Cross-source classification

Runs on top of stage 2 context — no raw data re-examination.

Same three-model architecture across graph boundaries:

- Model 1: cross-source naming and metadata similarity — works from existing node classifications
- Model 2: field value distribution similarity across graphs
- Model 3: adjudicates, produces confidence-scored cross-source references

Cross-source match quality compounds on per-source classification quality.

## Confidence thresholding and HITL

Above 80%: mapping auto-applied. Below 80%: two-tier resolution.

Build time HITL pass surfaces low-confidence mappings for human confirmation. This pass can be deferred to runtime. Deferred mappings surface to the caller LLM on first encounter during a live write. Confirmed mappings write back as high-confidence edges.

## Runtime self-mutation

Write failures — field rejections, type errors, unresolved references — feed back as classification signals. Confidence scores on affected nodes and edges revised. Strong enough signal triggers reclassification and re-evaluation of dependent cross-source relationships.

Graph converges on ground truth through operation.

## Schema drift

Periodic full rediscovery per source. New graph diffed against existing. Delta applied incrementally. Nodes with changed classifications flag dependent cross-source relationships for re-evaluation.

## Stage 4 — LLM tool interface

Graph exposed as a deterministic tool set. Agent reads and writes against the graph — no field mapping reasoning, no relationship inference. LLM ran at build time. Runtime is pure graph traversal.

## Open problems

At 80%+ confidence threshold, Manifold matches 98% of all fields across tested sources. The remaining 2% are concentrated in extremely custom or poorly named fields where neither naming context nor value statistics provide sufficient signal. These are the genuine hard cases — fields that would confuse a human engineer on first inspection as well.

The adjudication model has gone through multiple iterations and is relatively performant. Training signal quality is the primary lever — more diverse sources with known ground truth classifications improve it materially.

Upward walk edge cases around circular references and multiple valid ancestors. Handled with priority ordering on ancestor selection based on parent node semantic classification.

80% confidence threshold is empirically derived from observed false positive and false negative rates across tested sources. Expect it to shift as write feedback accumulates across diverse source combinations.

mfold.tools — early access open. Happy to go deep on any of the above.
