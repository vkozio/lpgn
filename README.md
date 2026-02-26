# LPGN: Locality-Preserving Graph Notation

**A graph serialization format aligned with Transformer self-attention**

[![Status: Specification](https://img.shields.io/badge/Status-Specification_v1.0-blue.svg)](#)
[![Domain: LLM & Graph AI](https://img.shields.io/badge/Domain-LLM%20%7C%20Graph%20Reasoning-orange.svg)](#)

Standard graph serialization formats (JSON, DOT, Cypher) were designed for algorithmic parsers, not for neural sequence models. When such formats are used directly as LLM prompts, connected nodes often become distant in the token sequence, which makes it harder for attention to recover the underlying topology and increases token usage.

Locality-Preserving Graph Notation (LPGN) is a compact, declarative graph notation that makes graph adjacency explicit in the token sequence, with the goal of improving how transformers read and generate graph-structured data.

---

## Problem: graph structure vs. token layout

In edge lists and JSON-based representations, node identifiers and edges are typically scattered across the text:
* indirect references through string IDs (for example, "source" / "target" fields)
* additional syntax noise around the actual graph elements
* repeated boilerplate for similar edge patterns

For LLMs, this means that adjacent nodes in the graph can end up far apart in the input sequence, and simple topologies may require many tokens to describe.

---

## LPGN in one example

Consider a simple business topology: a user submits an order, which is processed by both Risk and Finance in parallel, and then sent to Logistics.

JSON edge-list representation:

```json
{
  "nodes": ["User.Submit", "Risk.Check", "Finance.Hold", "Logistics.Ship"],
  "edges": [
    {"from": "User.Submit", "to": "Risk.Check"},
    {"from": "User.Submit", "to": "Finance.Hold"},
    {"from": "Risk.Check", "to": "Logistics.Ship"},
    {"from": "Finance.Hold", "to": "Logistics.Ship"}
  ]
}
```

Equivalent LPGN representation:

```lpgn
@User:Order.Submit -> @System:{Risk.Check, Finance.Hold} -> @Logistics:Ship
```

Here, nodes that are adjacent in the topology are also adjacent in the token sequence, and a multi-edge pattern is expressed in a single line.

---

## Design principles

LPGN follows two main principles:

### 1. Algebraic edge compression

Instead of listing every edge separately, LPGN uses combinatorial constructs (sets, cliques, chains) together with a flow operator to describe families of edges in one expression. Dense or bipartite patterns that would require many separate edge records in JSON can be written as a short algebraic fragment.

### 2. Textual–topological locality

LPGN encourages writing graphs so that adjacent nodes in the topology are also close in the text and can be evaluated left to right. Cycles are closed via explicit back-reference anchors, which avoid long-distance "GOTO"-style references while still allowing cyclic graphs.

---

## Quick syntax reference

| Operator | Concept | Description | LPGN example |
| --- | --- | --- | --- |
| {A, B} | Set | Parallel, disjoint nodes with no internal edges. | System:{Pack, Ship} |
| [A, B] | Clique | Fully connected internal subgraph. | [Review, Sign] |
| <A, B> | Chain | Sequential strict path. | <Step1, Step2> |
| @Actor: | Role | Semantic metadata anchor. | @System:Log |
| -> | Flow | Directed Cartesian flow from left side to right side. | A -> B |
| ?(...) | Gateway | Stochastic branching and merging. | ?(0.8 Pass \| 0.2 Fail) |
| ^Node | BackRef | Topological loop anchor for cyclic graphs. | Fail: ^Order.Draft |

---

## Intended use and properties

* Topology-aware prompting: keeps connected graph elements close in the token sequence, which can make it easier for LLMs to reconstruct local neighborhoods.
* Compact notation: compresses repeated edge patterns into algebraic constructs, reducing the prompt length for many graph topologies compared to naive edge lists.
* Canonical form (cLPGN): supports a canonical, lexicographically ordered subset intended for round-tripping between LPGN and edge-list representations.
* Human-readable: simple enough to be written and reviewed as plain text without specialized graph visualization tools.

---

## Repository structure

| Path | Contents |
|------|----------|
| [spec/](spec/) | Canonical specification. See spec/LPGN-Spec.md for the formal grammar, EBNF, and cLPGN. |
| parsers/ | Reference parsers (Python, Node) that parse LPGN into an AST or edge list. Planned. |
| converters/ | Converters from other formats to LPGN. Planned. |

The specification lives in spec/ rather than at the repository root or in docs/: this keeps the root clean as parsers for individual languages are added. The docs/ directory is reserved for project documentation (APIs, contributing guidelines), not the formal language specification.

---

## Parsers and converters (planned)

* Parsers: reference implementations in Python and Node that parse LPGN strings to an abstract syntax tree or edge list, and can be published as PyPI and npm packages.
* Converters to LPGN: for example, from Mermaid and DOT (Graphviz). Additional candidates include GraphML, GEXF, JSON (nodes and edges), CSV edge lists, and Cypher (read-only export for LLM context). Converters may live inside each parser package (for example, lpgn from-mermaid) or in a shared converters/ layer.

