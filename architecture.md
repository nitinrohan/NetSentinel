# NetSentinel — System Architecture (ML NIDS + RAG SOC Copilot)

> **One line:** network traffic → a trained model scores it → anything suspicious becomes an alert → a
> local LLM, grounded in a security knowledge base, explains that alert to an analyst in plain English.

This doc is the system-design view: the blocks, the decision logic, the orchestration, and the tools —
plus a reusable template at the bottom for the next project.

---

## 1. 10,000-ft view

```
                         ┌───────────────────────────────────────────────┐
                         │                 SOC ANALYST (user)             │
                         │     reads alerts + asks "what is this?"         │
                         └───────────────▲───────────────────────────────┘
                                         │ alerts + explanations
                                         │
                         ┌───────────────┴───────────────┐
                         │        FRONTEND (Streamlit)    │
                         │   alert table · explain panel  │
                         └───────▲────────────────▲───────┘
                                 │                │
                 labeled alerts  │                │ grounded explanation
                                 │                │
        ┌────────────────────────┴───┐     ┌──────┴──────────────────────────┐
        │   DETECTION ENGINE          │     │   RAG COPILOT ENGINE            │
        │  (supervised ML classifier) │     │  retrieve knowledge → generate  │
        └────────────▲────────────────┘     └──────▲───────────────────▲─────┘
                     │                               │                   │
          flow features                      security KB (vectors)   local LLM
                     │                               │                   │
        ┌────────────┴───────────┐        ┌──────────┴─────┐    ┌────────┴────────┐
        │  DATA SOURCE            │        │  Chroma vector │    │  Ollama         │
        │  CICIDS2017 CSV (now)   │        │  store         │    │  qwen2.5:7b     │
        │  scapy live capture     │        └────────────────┘    └─────────────────┘
        │  (Phase 4, later)       │
        └────────────────────────┘
```

Two engines, loosely coupled. The **Detection Engine** answers *"is this an attack?"* The **RAG Copilot
Engine** answers *"what does this attack mean and what do I do?"* They meet at the alert object.

---

## 2. The two planes (the core RAG idea)

RAG has an **offline/indexing plane** (run once, build the knowledge) and an **online/serving plane**
(run per request). Keeping these separate is the single most important system-design decision.

### 2a. OFFLINE — build the knowledge base (RAG Task 1)

```
   data/knowledge_base/*.md
            │
            ▼
      ┌───────────┐   split on "## " headings
      │  CHUNK    │   → one topic per passage
      └─────┬─────┘
            ▼
      ┌───────────┐   sentence-transformers
      │  EMBED    │   all-MiniLM-L6-v2 → 384-dim vector
      └─────┬─────┘
            ▼
      ┌───────────┐   persist to disk
      │  STORE    │   Chroma: {id, text, vector, source}
      └───────────┘
```

Runs when the knowledge changes, not per alert. Output: a populated Chroma collection.

### 2b. ONLINE — serve an alert (RAG Tasks 2–3)

```
   network flow row
          │
          ▼
   ┌─────────────┐   model2.predict_proba(x)[1]  → p
   │  SCORE      │
   └──────┬──────┘
          ▼
   ╔═════════════════════════╗
   ║  IF p >= threshold      ║───── no ──▶ drop / log as benign (no alert)
   ╚═══════════╤═════════════╝
               │ yes
               ▼
   ┌─────────────────────────┐   alert = {label, confidence, src, dst, ts}
   │  RAISE ALERT            │
   └───────────┬─────────────┘
               ▼
   ┌─────────────────────────┐   embed the alert text (SAME model as offline)
   │  EMBED QUERY            │
   └───────────┬─────────────┘
               ▼
   ┌─────────────────────────┐   collection.query(q, n_results=k)
   │  RETRIEVE top-k chunks  │   → k nearest security passages + sources
   └───────────┬─────────────┘
               ▼
   ╔═════════════════════════╗
   ║  IF best distance > τ   ║───── yes ─▶ "no KB match — raw alert only"
   ╚═══════════╤═════════════╝            (guard against irrelevant retrieval)
               │ no (good match)
               ▼
   ┌─────────────────────────┐   prompt = system + retrieved chunks + alert
   │  BUILD PROMPT           │
   └───────────┬─────────────┘
               ▼
   ┌─────────────────────────┐   ollama.chat(qwen2.5:7b-instruct)
   │  GENERATE explanation   │   "explain using ONLY these notes; cite sources"
   └───────────┬─────────────┘
               ▼
        grounded explanation  ──▶ Streamlit panel ──▶ analyst
```

---

## 3. Decision logic (the if/else blocks, spelled out)

```python
# per network flow
p = detector.predict_proba(flow)[1]          # attack probability 0..1

if p < THRESHOLD:                             # THRESHOLD is the Phase 3 dial
    log_benign(flow)                          # no alert, no LLM — cheap path
else:
    alert = make_alert(flow, label, p)

    # severity routing (false-positive cost awareness)
    if p >= 0.90:        severity = "HIGH"
    elif p >= THRESHOLD: severity = "MEDIUM"

    # retrieval quality guard
    hits = kb.retrieve(alert, k=3)
    if hits.best_distance > MAX_DISTANCE:     # nothing relevant in the KB
        explanation = fallback_raw_alert(alert)   # don't let the LLM invent
    else:
        explanation = llm.generate(alert, hits)   # grounded answer + citations

    push_to_ui(alert, severity, explanation)
```

Two guards worth noting (both are "senior" touches):
- **The threshold guard** keeps the LLM off benign traffic — you don't pay for generation on every packet.
- **The distance guard** refuses to generate when retrieval found nothing relevant, which is how you stop
  a RAG system from confidently explaining an alert it has no knowledge about (hallucination control).

---

## 4. Component → responsibility → tool

| Block | Responsibility | Tool / library | Plane |
|---|---|---|---|
| Data source | Produce flow feature rows | CICIDS2017 CSV now; `scapy` live later | input |
| Detection engine | Score flow → attack probability | scikit-learn RandomForest (`model2`), `joblib` | online |
| Threshold logic | Turn probability into alert/no-alert | plain Python (Phase 3 work) | online |
| Chunker | Split docs into single-topic passages | Python (`str.split`) | offline |
| Embedder | Text → 384-dim meaning vector | `sentence-transformers` (all-MiniLM-L6-v2) | both |
| Vector store | Hold vectors, answer nearest-neighbour | `chromadb` (PersistentClient) | both |
| Retriever | Alert → top-k relevant chunks | `chromadb.query` | online |
| Generator | Chunks + alert → explanation | `ollama` client + `qwen2.5:7b-instruct` | online |
| Frontend | Show alerts, show explanations | `streamlit` | online |
| Orchestrator | Wire score → alert → retrieve → generate → UI | Python glue (script/module) | online |

**Why these choices:** everything is local and $0 (Ollama replaces a paid API; Chroma needs no server;
MiniLM is tiny and fast on CPU). The embedding model is the one piece shared across both planes — it MUST
be identical offline and online, or the query and document vectors live in different spaces and retrieval
is meaningless.

---

## 5. Data contracts (what passes between blocks)

```
flow row        : dict/Series of ~78 numeric CICIDS features
alert           : {label, confidence, src_ip, dst_ip, timestamp}
chunk (stored)  : {id, text, embedding[384], metadata:{source}}
retrieval result: [{text, source, distance}, ...]  (k items)
explanation     : {summary, why_it_matters, recommended_actions, sources[]}
```

Defining these up front is what lets the two engines stay loosely coupled — the Copilot only needs the
`alert` contract, not the model internals.

---

## 6. Build order (phases → this architecture)

1. **Done:** Detection engine (Phases 1–2) + threshold logic (Phase 3).
2. **Now (Phase 7 fast-tracked):** offline plane — Task 1 chunk/embed/store.
3. **Next:** online plane — Task 2 retrieve, Task 3 generate.
4. **Then:** Task 4 frontend (Streamlit) + orchestrator glue.
5. **Later:** swap the data source from CSV to `scapy` live capture (Phase 4); cloud deploy (Phase 6).

---

## 7. Reusable template (curate your own next time)

Any system like this answers five questions — fill these blanks for the next project:

```
1. SOURCE     — where does raw input come from?        (CSV / live capture / price API / webhook)
2. DECISION   — what model/rule turns input into an    (classifier + threshold / trend rule /
                a signal worth acting on?                anomaly score)
3. KNOWLEDGE  — what docs ground the explanation,       (KB → chunk → embed → vector store)
                and how are they indexed (offline)?
4. SERVE      — per request: score → IF signal →        (retrieve top-k → prompt → LLM)
                retrieve → generate
5. SURFACE    — how does the human see it + what        (dashboard / notification / chat)
                guards stop bad output?                  (threshold guard, retrieval-distance guard)
```

For the **trading bot (Project #2)** that maps to: SOURCE = price API (CoinGecko) → DECISION = trend
rule / signal → KNOWLEDGE = market-concept notes → SERVE = retrieve + explain the signal →
SURFACE = notification ("consider withdrawing — here's why"). Same skeleton, different domain.
