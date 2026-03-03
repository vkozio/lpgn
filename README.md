## LPGN: Locality-Preserving Graph Notation

**A compact graph notation shaped for Transformer self-attention.**

LPGN is a tiny graph language where **graph adjacency is mirrored in token adjacency**. Instead of node/edge lists with scattered IDs, LPGN uses algebraic primitives (sets, cliques, chains) plus flow operators so that common topologies become short, contiguous spans of text that transformers can attend to as a whole.

The full formal specification (grammar, algebraic rules, canonical form) lives in the spec:

- **[LPGN Specification](spec/LPGN-Spec.md)** (version 0.1.0)

---

### Minimal examples

**Parallel processing and fan-in:**

```lpgn
@User:Order.Submit -> @System:{Risk.Check, Finance.Hold} -> @Logistics:Ship
```

**Stochastic gateway with retry:**

```lpgn
@@ meta (example "payment-flow" note "stochastic topology with retry")
@User:Order.Submit
-> @System:{Risk.Check, Finance.Hold}
-> ?Validation(0.95 Pass: <Process, Ship> | 0.05 Fail: ^Order.Submit)
-> @Customer:Notify
```

**Undirected / bipartite interaction:**

```lpgn
{User.Alice, User.Bob} ~ {Item.Laptop, Item.Phone}
```

Any line starting with `@@` is a **meta directive** and is ignored by the core LPGN grammar; tools are free to use it for ontologies, execution plans, or other metadata.

