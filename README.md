# LPGN: Locality-Preserving Graph Notation

**A compact graph notation shaped for Transformer self-attention**

LPGN is a tiny graph language where **graph adjacency is mirrored in token adjacency**.  
Instead of edge lists and scattered IDs, LPGN uses algebraic primitives (sets, cliques, chains) plus flow operators so that common topologies become short, contiguous fragments of text that transformers can attend to as a whole.

The full formal specification (EBNF, algebraic rules, canonical form) lives in  
**[`spec/LPGN-Spec.md`](spec/LPGN-Spec.md)**.

---

## Why: graph structure vs token layout

Most graph formats (JSON node/edge lists, DOT, Cypher) were designed for programmatic parsers:
- edges are records with string IDs in separate sections
- similar edge patterns are repeated verbosely
- additional syntax wraps the actual graph elements

For LLMs this means:
- adjacent nodes in the graph often land far apart in the token sequence
- simple topologies require many tokens to describe
- attention has to reconstruct locality through long-range ID matches instead of reading a dense structural phrase.

LPGN flips this: **you write the topology as one algebraic expression**, read strictly left-to-right.

---

## Core idea in one example

Business topology: a user submits an order, Risk and Finance process it in parallel, then Logistics ships it.

Classic JSON edge list:

```json
{
  "nodes": ["Order.Submit", "Risk.Check", "Finance.Hold", "Logistics.Ship"],
  "edges": [
    {"from": "Order.Submit", "to": "Risk.Check"},
    {"from": "Order.Submit", "to": "Finance.Hold"},
    {"from": "Risk.Check", "to": "Logistics.Ship"},
    {"from": "Finance.Hold", "to": "Logistics.Ship"}
  ]
}
```

The same topology in LPGN:

```lpgn
@User:Order.Submit -> @System:{Risk.Check, Finance.Hold} -> @Logistics:Ship
```

Adjacency is now explicit and local: one short fragment captures a bipartite fan-out and fan-in without repeating edge boilerplate.

---

## Quick syntax (topology only)

| Operator | Concept | Description | LPGN example |
| --- | --- | --- | --- |
| `{A, B}` | Set | Parallel, disjoint nodes with no internal edges. | `{Pack, Ship}` |
| `[A, B]` | Clique | Fully connected internal subgraph. | `[Review, Sign]` |
| `<A, B>` | Chain | Sequential strict path. | `<Step1, Step2>` |
| `@Actor:` | Role | Semantic role/actor anchor for a group. | `@System:Log` |
| `->` | Flow | Directed Cartesian flow from left side to right side. | `A -> B` |
| `~` | Undirected flow | Symmetric bipartite product, expands to both directions. | `A ~ B` |
| `?(...)` | Gateway | Weighted branching over groups. | `?(0.8 Pass: X \| 0.2 Fail: Y)` |
| `^Node` | BackRef | Loop back to a previously defined node. | `Fail: ^Order.Draft` |

Any line starting with `@@` is a **meta directive** and is ignored by the core grammar (you can use this channel for configs or ontologies).

---

## A few illustrative examples

### 1. Process with retry loop

```lpgn
@@ meta (example "payment-flow" note "stochastic topology with retry")
@User:Order.Submit
-> @System:{Risk.Check, Finance.Hold}
-> ?Validation(0.95 Pass: <Process, Ship> | 0.05 Fail: ^Order.Submit)
-> @Customer:Notify
```

Here you see:
- parallel Risk/Finance processing as a set `{Risk.Check, Finance.Hold}`
- stochastic gateway with explicit weights
- a retry loop via `^Order.Submit`.

### 2. Undirected / bipartite relationships

Users and products, treated as an undirected interaction graph:

```lpgn
{User.Alice, User.Bob}
~ {Item.Laptop, Item.Phone}
```

`~` is sugar: it expands to both `Users -> Items` and `Items -> Users` in the underlying edge list, while staying a single symmetric phrase for the model.

### 3. Hyperedge-style relation

Multiple roles accessing multiple resources through a single relation node:

```lpgn
<{Role.Admin, Role.Manager}, Perm.ReadWrite, {Resource.Billing, Resource.Reports}>
```

The middle node `Perm.ReadWrite` is a first-class relation; the surrounding sets give you an m:n “hyperedge” without leaving the core language.

---

## What LPGN is (and is not)

LPGN is:
- **a topology language**: directed graphs, undirected/bipartite sugar via `~`, back-references, stochastic gateways.
- **LLM-oriented**: designed so that a short, contiguous span of tokens encodes a meaningful neighborhood in the graph.
- **canonicalizable**: the spec defines cLPGN with deterministic ordering and biclique factorization for round-tripping with edge lists.

LPGN is **not**:
- a styling or rendering language (no built-in attributes or class blocks in the core).
- an ontology system: higher-level metadata should live in `@@` meta directives or separate layers.

For the exact grammar and algebraic rules, see the **[full specification](spec/LPGN-Spec.md)**.

