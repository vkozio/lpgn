---
title: LPGN — Locality-Preserving Graph Notation for LLM Self-Attention Optimization
description: Formal grammar for graph serialization optimized for LLM self-attention; topological primitives, roles, stochastic gateways; locality-preserving.
domain: research-lpgn
tags:
  - graph-notation
  - llm-optimization
  - formal-grammar
  - self-attention
status: specification
---

# LPGN: Locality-Preserving Graph Notation for LLM Self-Attention Optimization

**arXiv Category:** cs.CL (Computation and Language), cs.AI (Artificial Intelligence), cs.DS (Data Structures and Algorithms)

---

## Abstract

Graph structures are fundamental to knowledge representation, yet standard serialization formats (DOT, JSON, Cypher) suffer from high token redundancy and indirect referencing, severely degrading Large Language Model (LLM) reasoning and recall. This document introduces **Locality-Preserving Graph Notation (LPGN)**, a declarative, algebraically dense specification designed strictly around the Transformer Self-Attention paradigm. LPGN enforces a topological-lexical isomorphism where graph adjacency equals token adjacency. By replacing verbose edge lists with combinatorial primitives, bipartite semantic chains, and stochastic gateways, LPGN achieves extreme token compression. Furthermore, the specification defines a Canonical Form (cLPGN) ensuring a strict 1:1 bidirectional mapping between topology and text, maximizing semantic recall and predictive accuracy in LLMs.

---

## 1. Design Philosophy & LLM Attention Constraints

Standard graph serialization relies heavily on indirect referencing (defining nodes in one block and linking their IDs in another). This indirectness forces the LLM's attention mechanism to disperse weights across long context windows, leading to hallucinated edges and degraded recall. LPGN solves this via three strict constraints:

* **Strict Textual-Topological Locality (Anti-Aliasing):** Variables, modular imports, and programmatic indirect references are forbidden. The graph must be evaluated "in-place". If nodes are adjacent in the topology, their tokens must be structurally adjacent in the text.
* **Bipartite Semantic Edges (No Edge-Level Labels):** Inserting metadata explicitly between two nodes (e.g., `A -[creates]-> B`) increases token distance and weakens mutual attention. If a relationship must be named, the topology must be modeled as a bipartite chain where the action becomes a structural node: `<Actor, Action.Create, Object>`. The same pattern generalizes to hyperedges and factor-graph style relations: a relation node connects sets of sources and targets (e.g., `<{A, B}, Relation.Assign, {C, D}>`).
* **Cyclic Topology via Zero-Logic Back-References:** Real-world graphs contain loops. To represent them without procedural constructs (like GOTO or loops), LPGN uses a purely topological anchor `^` to close a cycle, pointing back to a historically defined node without breaking the left-to-right reading flow.

---

## 2. Expressiveness Scope

LPGN is explicitly designed to represent **Directed Graphs** with node labels, semantic metadata (actor roles), and stochastic branching, and to derive **undirected** and **bipartite** structures as thin sugar over the same primitives.

**Supported:**

* Directed Acyclic Graphs (DAGs) and Directed Cyclic Graphs (via back-references).
* Undirected graphs and symmetric relations via the `~` undirected flow operator, which expands to a pair of opposite flows (`S1 -> S2` and `S2 -> S1`) in the underlying edge list.
* Bipartite graphs (two disjoint node partitions with only cross-part edges) modeled via Sets/Cliques combined with `->` or `~` flows.
* Hyperedge-like relations (m:n connections) encoded via explicit relation nodes in bipartite chains (e.g., `<{A, B}, Relation.X, {C, D}>`), preserving locality.
* Role-based metadata anchoring.
* Weighted branching gateways (Float = weight/metric; the parser builds edges with these weights and does not interpret their meaning).

**Strictly Not Supported (Out of Scope):**

* Multigraphs (multiple distinct labeled edges between the exact same ordered pair of nodes; if different semantics are needed, they must be lifted into separate relation nodes).
* Custom inline edge labels or properties (must be converted to node chains).

LPGN core does not define node attributes, styling blocks, or ontological dictionaries as part of the topology language. Any non-topological metadata (visual styles, domain ontology, execution configuration, documentation) MUST be encoded outside the main Expression and interpreted by downstream tools.

To support such metadata in a lightweight and implementation-agnostic way, LPGN reserves **line-level meta directives** starting with `@@`. A meta directive is any single line whose first two characters are `@@`; the remaining payload is opaque to LPGN and MAY use any syntax. Core LPGN parsing MUST treat all such lines as out-of-band metadata that does not participate in topology or edge expansion; tools MAY consume them as ontological dictionaries, renderer configuration, execution-plan annotations, or other meta-configuration without affecting the graph semantics. Any number of meta directives MAY appear before or after the main Expression; they are not part of the LPGN graph itself.

In practice, implementations usually pick a **simple, uniform payload format** for `@@` so it is easy to parse. Recommended patterns:

- `@@ TransferNode {"qty": 7, "cost": 0.1}` – easily distinguishable `DirectiveName JSONPayload` format for tooling
- `@@ DictOntology (ontology (Class User (kind "actor")))` – `DirectiveName` with an S-expression payload

LPGN does not interpret these payloads; tools MAY define any schemas they need while relying on the fact that `@@` lines are ignored by topology parsing.

---

## 3. Quick Reference Syntax

| Operator | Name | Function | Example |
| --- | --- | --- | --- |
| `Domain.X` | Namespace | Groups entities by domain | `Finance.Invoice` |
| `{A, B}` | Set | Parallel, disjoint nodes. No internal edges | `{Pack, Ship}` |
| `[A, B]` | Clique | Fully connected nodes (all pairs) | `[Review, Sign]` |
| `<A, B>` | Chain | Sequential path | `<Step1, Step2>` |
| `@Actor:` | Role Anchor | Binds an actor to a node/group | `@System:Log` |
| `->` | Flow | Directed Cartesian product | `A -> B` |
| `~` | Undirected Flow | Symmetric bipartite product (expands to `A -> B` and `B -> A`) | `A ~ B` |
| `?(...)` | Gateway | Weighted branching (Float = weight/metric per branch) | `?(0.8 A | 0.2 B)` |
| `^Node` | Back-Reference | Topological loop anchor | `Fail: ^Draft` |

---

## 4. Formal Grammar (EBNF)

To guarantee unambiguous parsing, LPGN adheres to the following Extended Backus-Naur Form. A document contains a single top-level Expression; any lines starting with `@@` are meta directives and are ignored by this grammar.

```ebnf
Document    ::= Expression

Expression  ::= Group ( RelationalOp Group )*
Group       ::= RoleAnchor? ( Node | Primitive | Gateway | BackRef | "(" Expression ")" )
Primitive   ::= Set | Clique | Chain
Set         ::= "{" NodeList "}"
Clique      ::= "[" NodeList "]"
Chain       ::= "<" NodeList ">"
Gateway     ::= "?" Identifier "(" WeightedBranch ( "|" WeightedBranch )* ")"
WeightedBranch ::= Float Identifier ":" Group
BackRef     ::= "^" Identifier
NodeList    ::= Node ( "," Node )*
Node        ::= Identifier ( "." Identifier )?
RoleAnchor  ::= "@" Identifier ":"
RelationalOp::= "->" | "~"
Identifier  ::= [a-zA-Z_] [a-zA-Z0-9_]*
Float       ::= [0-9]+ "." [0-9]+

```

In Gateway, each WeightedBranch is Float (weight/metric) plus branch label and Group; the parser builds edges with these weights and does not interpret their meaning. RelationalOp: `->` (Flow) and `~` (UndirectedFlow); `S1 ~ S2` MUST expand to the symmetric pair of flows `S1 -> S2` and `S2 -> S1` in the underlying edge list. RoleAnchors are purely semantic labels attached to groups; they do not affect algebraic expansion beyond labeling the produced nodes.

---

## 5. Algebraic Expansion Rules (Unfolding)

Parsing an LPGN string into a standard edge list is an O(n) operation over the input length, though the generated edge count can reach O(n²). Expansion relies on set theory. Let S1, S2, and S3 be sets of nodes produced by the respective grammar groups.

* **Set `{A, B, C}`:** Generates vertices but zero internal edges.

* **Clique `[A, B, C]`:** Generates a fully connected subgraph (excluding self-loops).

* **Chain `<A, B, C>`:** Connects elements sequentially.

* **Flow `S1 -> S2`:** Generates a directed bipartite Cartesian product.

* **Undirected Flow `S1 ~ S2`:** Generates a symmetric bipartite Cartesian product, equivalent to `S1 -> S2` plus `S2 -> S1`. Implementations may store undirected edges explicitly or as pairs of directed edges; canonicalization MUST treat `S1 ~ S2` and `S2 ~ S1` as equivalent.

* **Back-Reference `^Id`:** Resolves to the first node with that identifier appearing earlier in the left-to-right topology. If none exists, the graph is invalid (Unresolved Reference).

---

## 6. Canonical Form (cLPGN) and Isomorphism

Because commutative primitives (`{}` and `[]`) allow multiple valid string representations for the same topology, standard LPGN is polymorphic. To achieve strict structural isomorphism (one unique graph = one unique string) for optimized LLM fine-tuning, the **Canonical LPGN (cLPGN)** enforces these deterministic constraints:

1. **Topological Leveling:** Serialize the graph in breadth-first order over the expanded topology (sources first, then nodes at distance 1, 2, …). This keeps related nodes close in the token stream and avoids deep branches fragmenting attention.

2. **Lexicographical Ordering:** Nodes within any Set `{}` or Clique `[]` MUST be sorted alphanumerically. For symmetric flows using `~`, implementations SHOULD treat `S1 ~ S2` and `S2 ~ S1` as equivalent and normalize to a stable textual ordering (e.g., by lexicographically comparing the serialized forms of `S1` and `S2`).

   *Invalid:* `{B, A} -> C`  
   *Valid (cLPGN):* `{A, B} -> C`

3. **Greedy Biclique Factorization:** Redundant edges must be factored into the largest possible Cartesian products (Flows) before serialization. If edges A→C and B→C exist, they must be represented as `{A, B} -> C`, not as disjoint chains. Ties are broken by choosing subsets that contain the lexicographically earliest nodes.

---

## 7. Author guidelines for LLM-friendly graphs

Authors should keep attention and context limits in mind:

* **Clique size:** Prefer small, cohesive cliques. Large cliques (e.g. dozens of nodes) produce many bidirectional edges and dilute attention; use Sets and Flows for broad fan-out instead.
* **Modularity:** For large graphs, use Domain namespaces to group nodes and keep Back-References short. Long-distance back-references increase the span the model must attend over.

---

## 8. Example: Stochastic Topology with Cyclic Recovery

**Visualizing a complex payment flow with a retry loop:**

```lpgn
@@ meta (example "payment-flow" note "stochastic topology with retry")
@User:Order.Submit 
-> @System:{Risk.Check, Finance.Hold} 
-> ?Validation(0.95 Pass: <Process, Ship> | 0.05 Fail: ^Order.Submit) 
-> @Customer:Notify
```

*Note: The LLM reads this strictly left-to-right. The `^Order.Submit` back-reference securely closes the topology without interrupting the forward semantic attention for the success path. The leading `@@ meta` line is optional and illustrates how a meta directive can annotate the entire example without affecting its graph semantics.*

