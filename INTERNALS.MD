# Manifold – The data farbic for agents that act

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
