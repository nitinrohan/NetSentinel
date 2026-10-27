# NetSentinel — ML Network Intrusion Detection + RAG SOC Copilot

**Detect and explain.** NetSentinel applies machine learning to network traffic to flag
malicious activity (port scans, DDoS, brute-force, etc.), then uses a Retrieval-Augmented
Generation (RAG) "SOC Copilot" to explain each alert in plain English — grounded in a
security knowledge base. Built to bridge a data science / analytics background into network
security & SOC-style detection work.

_Stack: Python · scikit-learn · sentence-transformers · Chroma · Ollama (local, $0)._

## Why this project
Security Operations Centers (SOCs) and detection engineering teams drown in network and
log data. This project applies ML/analytics to that data to automatically flag intrusions —
the intersection of data science and network defense.

## Roadmap
- [x] **Phase 1 – Foundations + first model** (NSL-KDD): network flows & attack categories;
      Random Forest classifier evaluated on the official test set (precision 0.97, recall 0.61 —
      first look at the generalization gap).
- [x] **Phase 2 – Realistic data** (CICIDS2017): ~2.5M flows across 7 days; recall measured
      *per attack type*; cross-day tests expose held-out attacks (e.g. PortScan) dropping to
      ~0% recall — the limit of supervised detection on novel attacks.
- [x] **Phase 3 – Detection engineering**: confusion matrices, precision-recall curves, and
      decision-threshold tuning; choosing operating points by false-positive cost.
- [~] **Phase 7 – SOC Copilot (RAG, $0)** *(fast-tracked, in progress)*: a Retrieval-Augmented
      Generation assistant that explains each flagged alert in plain English using a security
      knowledge base (attack descriptions + MITRE ATT&CK + remediation). Fully local:
      Ollama (LLM) + sentence-transformers (embeddings) + Chroma (vector store). **Done:**
      knowledge base built (chunk → embed → store, semantic retrieval verified).
- [ ] **Phase 4 – Live capture**: sniff real traffic with scapy, extract features, score it live.
- [ ] **Phase 5 – Portfolio polish**: dashboard, writeup, GitHub README.
- [ ] **Phase 6 – Cloud integration ($0)**: deploy the detector as a service and ingest
      real cloud security logs (public AWS VPC Flow Logs / CloudTrail samples), using
      LocalStack + free tiers only. No paid cloud resources.

> See [`architecture.md`](architecture.md) for the system design (detection engine + RAG copilot).

## Structure
```
data/raw/         # downloaded datasets (gitignored)
data/processed/   # cleaned/feature-engineered data
notebooks/        # exploratory analysis
src/              # reusable pipeline code
models/           # trained models
reports/          # figures, findings, writeups
```

## Setup
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Getting the data
The NSL-KDD dataset is not committed (it's large). Download it into `data/raw/`:
```bash
mkdir -p data/raw && cd data/raw
BASE="https://raw.githubusercontent.com/jmnwong/NSL-KDD-Dataset/master"
curl -sSL -o "KDDTrain+.txt" "$BASE/KDDTrain+.txt"
curl -sSL -o "KDDTest+.txt"  "$BASE/KDDTest+.txt"
```

## Concepts learned
Networking: TCP/IP, ports, protocols (TCP/UDP/ICMP), flows vs packets.
Security: intrusion types, detection evaluation, false-positive tradeoffs.
