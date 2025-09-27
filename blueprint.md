# Blueprint

**Knowledge (K)**, **Perception (P)**, and **Action (A)** as **one typed, navigable hypergraph** with stable IDs, explicit+implicit links, and a closed loop of retrieve → reason → act → verify → update.

---

## 1) Goals & principles

* **One substrate:** unify artifacts (K), telemetry (P), and actuators (A) in a single graph.
* **Typed links + stable identity:** every entity and span has a durable ID.
* **Multi-resolution:** byte/token ↔ element ↔ component ↔ system.
* **Deterministic core, learned edges:** explicit graph ops are deterministic; semantics/ranking can be learned.
* **Closed loop:** actions produce signals that update the graph and future decisions.

---

## 2) Core data model

### Node (canonical schema)

```json
{
  "id": "ns:kind:unique-key[@version|@timestamp]",
  "layer": "K|P|A",
  "kind": "outcome|doc|component|dataset|item|event|tool|policy|actor|...",
  "props": { "name": "...", "lang": "py", "uri": "...", "span": {"start":0,"end":10}, "meta": {...} },
  "provenance": { "source": "ingester_name", "hash":"...", "ts":"2025-09-27T10:00:00Z" },
  "acl": { "owner":"team-x", "labels":["pii:none"] }
}
```

### Edge (canonical schema)

```json
{
  "src": "node-id",
  "dst": "node-id",
  "kind": "defines|references|calls|depends-on|tests|documents|configured-by|produces|consumes|violates|allowed-by|latent-sim|...",
  "weight": 1.0,
  "attrs": { "conf": 0.92, "evidence": ["uri#anchor", "..."] },
  "valid": { "since":"...", "until": null }
}
```

### Layers

* **K (Knowledge):** versioned artifacts: specs, code/AST, schemas, docs, configs, standards.
* **P (Perception):** events & summaries: logs, metrics, test results, feedback, incidents.
* **A (Action):** tools & operations: compilers, workflow steps, deployers, patchers, API calls, policies (as executable checks).

**Identity & versioning:** `id = namespace:kind:stable-key@commit|@ts`. Keep *immutables* for historical snapshots; add `alias` edges for latest.

---

## 3) Type registry (extensible)

Maintain a registry to validate what edges are legal between kinds.

```yaml
node_kinds:
  - doc: {layer: K}
  - component: {layer: K}
  - dataset: {layer: K}
  - item: {layer: K}
  - outcome: {layer: K}
  - event: {layer: P}
  - metric: {layer: P}
  - tool: {layer: A}
  - policy: {layer: A}

edge_kinds:
  defines:        {src: [doc,component], dst: [item,outcome,component]}
  references:     {src: [doc,item,component], dst: [component,dataset,doc]}
  depends-on:     {src: [component,workflow], dst: [component,dataset,tool]}
  tests:          {src: [item,tool,policy],   dst: [component,outcome]}
  documents:      {src: [doc],                dst: [component,item,outcome]}
  produces:       {src: [tool,component],     dst: [event,metric,dataset]}
  consumes:       {src: [tool,component],     dst: [dataset,config]}
  configured-by:  {src: [component,tool],     dst: [doc,config]}
  violates:       {src: [component,change],   dst: [policy]}
  allowed-by:     {src: [component,change],   dst: [policy]}
  latent-sim:     {src: ["*"],                dst: ["*"]}
```

---

## 4) Storage & indexing

* **Graph store:** property graph (Neo4j/JanusGraph) or in-proc (NetworkX) for prototyping.
* **Vector index:** FAISS/Qdrant/Milvus for embeddings to create `latent-sim` edges.
* **Blob store:** raw artifacts (docs, files, binaries) with URI pointers in nodes.
* **Cache/indexes:** inverted index for text (BM25), and per-kind secondary indexes.

---

## 5) Build pipeline

1. **Ingest K:** parse/normalize artifacts; extract entities & spans; upsert nodes/edges; compute embeddings; write `defines/references/depends-on/...` edges.
2. **Ingest P:** stream events (tests, logs, metrics); aggregate into `event/metric` nodes; link with `produces/observed-in/tests` edges.
3. **Register A:** declare tools with **capabilities** and **schemas** (see §8). Link tools to targets with `operates-on` or `supports` edges.
4. **Latent links:** periodically compute/top-k `latent-sim` edges per node kind.
5. **Governance:** stamp provenance, ACLs, PII labels; run schema/edge validators.

---

## 6) Retrieval & reasoning

### Query struct (normalized)

```json
{
  "task": "plain text or DSL",
  "targets": ["node-id", "..."],   // optional seeds
  "constraints": { "time_budget": 600, "policy_labels": ["pii:none"] }
}
```

### Algorithm (deterministic core + learned scoring)

1. **Seed selection:** BM25 over text attrs + ANN in the vector index; union with explicit `targets`.
2. **Graph walk:** Personalized PageRank (PPR) or bounded multi-hop over a **typed edge policy**:

   ```python
   EDGE_W = {"defines":1.0,"depends-on":0.9,"references":0.8,"tests":0.8,
             "documents":0.6,"configured-by":0.6,"latent-sim":0.4}
   scores = PPR(G, personalization=seeds, edge_weights=EDGE_W, alpha=0.15)
   ```
3. **Candidate set:** take top-N by score; optionally **cross-encoder** rerank against `task`.
4. **Packing:** budget-aware selection of snippets/artifacts prioritized by kind
   (e.g., *definitions → callers/consumers → tests/policies → docs*).

Output: a **context pack** (ordered nodes + excerpts + provenance).

---

## 7) Planning (decision layer)

Represent a plan as a typed list of intents with preconditions and expected signals.

```json
{
  "plan_id": "pl-123",
  "intents": [
    {
      "action": "apply_patch|run_workflow|reconfigure|generate_doc|assign_item|backfill|...",
      "target": "node-id",
      "params": {"..."},
      "preconditions": [{"kind":"policy_pass","policy":"policy:foo"}],
      "expected_signals": [{"kind":"test_pass","node":"item:bar"}]
    }
  ]
}
```

A planner (rule-based or LLM) proposes the plan; a validator checks **types, policies, ACLs, and preconditions**.

---

## 8) Actuation (safe execution)

### Tool capability schema

```json
{
  "id": "tool:deploy",
  "kind": "tool",
  "layer": "A",
  "capabilities": [
    {
      "name":"deploy_component",
      "params_schema": { "component_id":"string", "env":{"enum":["staging","prod"]} },
      "effects": ["produces:event:deploy","updates:component:state"],
      "guards": ["policy:change_approved","policy:rbac_ok"],
      "dry_run": true
    }
  ]
}
```

**Executor contract**

1. Resolve `target` to concrete artifacts.
2. Evaluate **guards** (policies, RBAC, rate limits). If any fail → stop with explanation.
3. Run **dry-run** if supported; attach diff/previews to the plan.
4. Apply action; capture raw outputs and structured **signals** (events/metrics/tests).
5. Emit `produces` edges and update nodes; trigger incremental re-indexing.

---

## 9) Feedback & learning

* **Telemetry ingestion:** convert raw outputs into `event/metric` nodes; link with `produces/tests/observed-in`.
* **State estimation:** learned models may update per-node priors (e.g., mastery or risk).
* **Confidence:** compute confidence & residual risk from (coverage, policy gates, history).
* **Edge updates:** reinforce/decay `latent-sim` edges; update `violates/allowed-by` per policy outcomes.

---

## 10) APIs (service surface)

**Graph API**

* `upsert_node(node)`, `upsert_edge(edge)`, `subgraph(seed_ids, hops, edge_policy)`
* `neighbors(id, edge_filter)`, `find(pattern)`, `ppr(seeds, edge_weights)`

**Retrieval API**

* `build_context(task, targets?, constraints?) -> ContextPack`

**Planning API**

* `propose_plan(ContextPack) -> Plan`
* `validate_plan(Plan) -> {ok|errors}`

**Execution API**

* `execute(Plan) -> ExecutionReport`
* `dry_run(Plan) -> PreviewReport`

**Telemetry API**

* `ingest_events(list[Event])`, `summarize(window, kinds)`

**Policy API**

* `evaluate(policy_id, targets) -> {pass|fail, evidence}`

---

## 11) Reference loop (pseudocode)

```python
def run_task(task, targets=None, constraints=None):
    ctx = build_context(task, targets, constraints)       # §6
    plan = propose_plan(ctx)                               # §7 (can be LLM+rules)
    ok, errs = validate_plan(plan)                         # types, ACL, policy
    if not ok: return {"status":"blocked", "errors": errs}

    preview = dry_run(plan)                                # diffs, impact analysis
    if preview.blocked: return {"status":"blocked", "errors": preview.errors}

    report = execute(plan)                                 # §8
    ingest_events(report.events)                           # §9
    return {"status":"done", "report": report}
```

---

## 12) Minimal Python scaffold (prototype)

```python
import networkx as nx
from collections import defaultdict

G = nx.MultiDiGraph()

def add_node(id, layer, kind, **props): G.add_node(id, layer=layer, kind=kind, **props)
def add_edge(a, b, kind, weight=1.0, **attrs): G.add_edge(a, b, kind=kind, weight=weight, **attrs)

# Example: K
add_node("doc:spec:auth@v1", "K", "doc", uri="s3://specs/auth_v1.md")
add_node("component:auth-service@abc123", "K", "component")
add_edge("doc:spec:auth@v1", "component:auth-service@abc123", "documents")

# Example: P
add_node("event:test:auth_smoke#2025-09-27T10:00Z", "P", "event", result="pass")
add_edge("event:test:auth_smoke#2025-09-27T10:00Z", "component:auth-service@abc123", "tests")

# Example: A (tool)
TOOLS = {
  "deploy": {
    "capabilities": {"deploy_component": {"params":["component_id","env"], "guards":["policy:change_approved"]}}
  }
}

def ppr(seeds, edge_w):
    pers = defaultdict(float); [pers.__setitem__(s,1.0/len(seeds)) for s in seeds]
    # weight per edge-kind via an edge weight function
    G2 = nx.DiGraph()
    for u,v,k,d in G.edges(keys=True, data=True):
        w = edge_w.get(d.get("kind"), 0.0)
        if w>0: G2.add_edge(u,v,weight=w)
    return nx.pagerank(G2, alpha=0.85, personalization=pers, weight="weight")

def build_context(task, targets=None, constraints=None):
    seeds = set(targets or [])  # plus BM25/ANN results in production
    scores = ppr(seeds or list(G.nodes)[:1], {"documents":0.6,"tests":0.8,"depends-on":0.9,"latent-sim":0.4})
    ranked = [n for n,_ in sorted(scores.items(), key=lambda kv: kv[1], reverse=True)]
    return {"task":task, "nodes": ranked[:100]}

def execute(plan):
    # stub: run tools and produce events
    return {"events":[{"id":"event:run#1","kind":"event","result":"ok"}]}
```

Swap **NetworkX** for a production graph DB, and plug in **BM25 + embeddings** in `build_context`.

---

## 13) Safety, governance, audit

* **RBAC + labels:** enforce access by node/edge labels (e.g., `pii:*`, `env:prod`).
* **Policy gates:** all actions must pass policies (`allowed-by/violates` edges recorded).
* **Dry-run & impact analysis:** default-on for destructive actions.
* **Provenance & replay:** store inputs, context packs, plans, tool outputs for audit.
* **Reproducibility:** pin versions (`@commit/@ts`) for every node involved in a plan.

---

## 14) Metrics & evaluation

* **Coverage:** % of entities with explicit edges; % tasks retrieving tests/policies/docs.
* **Retrieval quality:** hit@k, MRR of target nodes in context packs.
* **Action quality:** success rate, rollback rate, policy violations, time-to-safe-apply.
* **Learning:** improvement deltas tied to signals (e.g., fewer regressions, higher mastery).
* **Cost/latency:** graph walk + rerank + tool runtime.

---

## 15) Scaling notes

* **Incremental indexing:** watch sources; re-embed and update only changed nodes.
* **Sharding:** by namespace or layer; keep `latent-sim` local + a thin global ANN.
* **Caching:** memoize frequent subgraphs and context packs by (task, target, commit).
* **Batching:** coalesce telemetry updates; defer heavy recompute to background jobs.

If you want, tell me your domain and I’ll instantiate this blueprint into a concrete node/edge registry and a 1-week implementation plan.
