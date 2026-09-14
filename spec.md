# CitationGuard BD — Project Specification

> **Type:** Multi-agent RAG system with evaluation + security pipeline  
> **Domain:** Legal aid, healthcare, disaster relief for Bangladeshi NGOs  
> **Tagline:** A self-correcting, citation-grounded AI assistant for under-resourced NGOs — where a wrong answer has real consequences.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Goals and Non-Goals](#2-goals-and-non-goals)
3. [Architecture Overview](#3-architecture-overview)
4. [Component Specifications](#4-component-specifications)
5. [Modal Inference Layer](#5-modal-inference-layer)
6. [RAG Pipeline Spec](#6-rag-pipeline-spec)
7. [Evaluation Spec](#7-evaluation-spec)
8. [Security Spec](#8-security-spec)
9. [Data Sources](#9-data-sources)
10. [Tech Stack](#10-tech-stack)
11. [API Contracts](#11-api-contracts)
12. [Deployment Spec](#12-deployment-spec)
13. [Build Order](#13-build-order)
14. [Known Constraints](#14-known-constraints)
15. [Success Criteria](#15-success-criteria)

---

## 1. Problem Statement

NGOs operating in Bangladesh — legal aid clinics, community health programs, disaster relief teams — need to answer complex, cross-domain queries from vulnerable clients. They cannot afford commercial LLM APIs. Existing tools hallucinate without citations, operate only in English, or require expensive infrastructure.

A garment worker illegally dismissed needs legal rights guidance **and** health coverage information in the same query. A flood victim needs both relief eligibility criteria and land dispute guidance. No single-agent, single-domain system can serve this reliably.

This system provides a hosted multi-agent AI assistant that:
- Runs entirely on free-tier infrastructure (Modal + Render)
- Answers in Bengali and English
- Cites every claim to a source document
- Refuses to answer when confidence falls below a safety threshold
- Serves cross-domain queries through parallel specialist agents

---

## 2. Goals and Non-Goals

### Goals

| ID | Goal |
|----|------|
| G1 | Answer legal, health, and disaster queries grounded in Bangladesh-specific documents |
| G2 | Support Bengali-language input and output |
| G3 | Block uncited or low-confidence answers before delivery |
| G4 | Run within Modal free tier ($30/month) and Render free tier (512MB RAM) |
| G5 | Pipeline is data-agnostic — any NGO can swap in their own documents |
| G6 | Every pipeline decision is documented and benchmarked (portfolio requirement) |

### Non-Goals

| ID | Non-Goal |
|----|----------|
| NG1 | Not a real-time system — latency of 15–40s is acceptable |
| NG2 | Not a general-purpose chatbot — only answers from uploaded documents |
| NG3 | Not a substitute for legal or medical professionals — always recommend consultation |
| NG4 | No user authentication in v1 — single-tenant, NGO admin manages documents |
| NG5 | No fine-tuning in v1 — prompt engineering and RAG only |
| NG6 | No document versioning in v1 |

---

## 3. Architecture Overview

```
User Query (Bengali / English)
            │
            ▼
┌─────────────────────────────────────────┐
│           Render — Application Layer     │
│                                         │
│  Language Handler                       │
│  (detect + translate Bengali ↔ English) │
│            │                            │
│            ▼                            │
│       Router Agent                      │
│  (classify domains to invoke)           │
│            │                            │
│    ┌───────┼───────┐                    │
│    ▼       ▼       ▼                    │
│  Legal  Health  Disaster                │
│  Agent   Agent   Agent                  │
│    │       │       │  (each calls Modal)│
│    └───────┼───────┘                    │
│            ▼                            │
│     Synthesis Agent                     │
│  (merge + resolve conflicts)            │
│            │                            │
│            ▼                            │
│   Citation Trust Gate                   │
│   (RAGAS faithfulness eval)             │
│            │                            │
│     Block  OR  Deliver                  │
└─────────────────────────────────────────┘
            │ HTTP calls
            ▼
┌─────────────────────────────┐
│   Modal — Inference Layer   │
│                             │
│   POST /embed    (CPU)      │
│   POST /rerank   (CPU)      │
│   POST /generate (T4 GPU)   │
└─────────────────────────────┘
```

**Separation of concerns:**
- **Modal** owns all model weights and inference. It exposes three HTTP endpoints.
- **Render** owns all application logic — orchestration, RAG pipeline, evaluation, routing, response delivery.
- **Qdrant Cloud** (free tier) owns vector persistence. Three namespaces: `legal_bd`, `health_bd`, `disaster_bd`.

---

## 4. Component Specifications

### 4.1 Language Handler

**Responsibility:** Detect language; translate Bengali → English before retrieval; translate final response English → Bengali if needed.

| Property | Value |
|----------|-------|
| Model | `facebook/nllb-200-distilled-600M` |
| Hardware | CPU (Render container) |
| Size | ~1.2GB — pre-baked into Docker image |
| Input | Raw user query string |
| Output | `{ original, translated, language: "bn" or "en" }` |

If input is already English: pass through unchanged. Translation is only applied at query-in and response-out boundaries.

---

### 4.2 Router Agent

**Responsibility:** Classify the query into one or more domains and return which specialist agents to invoke.

| Property | Value |
|----------|-------|
| Model | Mistral 7B via Modal `/generate` |
| Hardware | Render (calls Modal) |
| Output | `{ domains: ["legal", "health", "disaster"], confidence: float }` |

**Routing rules:**
- If one domain with confidence ≥ 0.75 → invoke that agent only
- If two domains or confidence < 0.75 → invoke all matching agents
- Default fallback: invoke all three agents

**Classification system prompt:**
```
You are a query classifier for a Bangladesh NGO assistant.
Given a query, return a JSON object with the domains it belongs to.
Domains: legal, health, disaster. A query can belong to more than one.
Return ONLY valid JSON. No explanation.
Example: { "domains": ["legal", "health"], "confidence": 0.82 }
```

---

### 4.3 Specialist Agents

All three agents share the same RAG pipeline structure. They differ only in system prompt and vector store namespace.

#### Legal Agent

| Property | Value |
|----------|-------|
| Namespace | `legal_bd` |
| Corpus | Bangladesh Labor Act 2006, VAWC Act 2000, NLASO eligibility, Land Survey Act |

**System prompt:**
```
You are a Bangladesh legal aid assistant. Answer only from the provided
context documents. Always cite the Act name and section number for every
claim. If the documents do not contain a reliable answer, say explicitly:
"I cannot find reliable legal guidance on this in the provided documents."
Do not speculate. Do not answer from general knowledge.
```

---

#### Health Agent

| Property | Value |
|----------|-------|
| Namespace | `health_bd` |
| Corpus | BRAC CHW field manuals, DGDA essential medicines list, Bangladesh National Health Policy, WHO Cox's Bazar guidelines |

**System prompt:**
```
You are a community health worker assistant for Bangladesh NGOs.
Be conservative with medical information. For any dosage, treatment, or
diagnosis, always include: "This should be confirmed with a licensed physician."
Cite the document title and section for every claim. Never speculate on
dosages or contraindications not present in the provided documents.
```

---

#### Disaster Relief Agent

| Property | Value |
|----------|-------|
| Namespace | `disaster_bd` |
| Corpus | BDRCS cyclone shelter protocols, SPARRSO flood guidelines, DDM relief distribution criteria |

**System prompt:**
```
You are a disaster relief coordinator assistant for Bangladesh.
Prioritize safety information above all else. Include nearest shelter
types or relief eligibility criteria where available in the documents.
Cite source document and section for every claim. If safety information
is unavailable in the documents, direct the user to call 1090 (Bangladesh
Disaster Management hotline).
```

---

### 4.4 Synthesis Agent

**Responsibility:** Merge outputs from multiple specialist agents into one coherent, non-contradictory response.

**Logic:**
- If only one agent was invoked: pass its answer through directly.
- If multiple agents invoked: call Modal `/generate` with merged chunks and synthesis prompt.
- If agents contradict each other: surface both explicitly.

**Synthesis system prompt:**
```
You are merging answers from multiple specialist agents into one response.
Preserve all citations. If sources contradict each other, present both
answers clearly and flag the conflict:
"Note: [Legal / Health] documents give different guidance on this point."
Do not invent information. Do not drop citations.
```

---

### 4.5 Citation Trust Gate

**Responsibility:** Score the synthesized answer against retrieved chunks. Block delivery if score falls below threshold.

| Property | Value |
|----------|-------|
| Tool | RAGAS `faithfulness` scorer |
| Delivery threshold | faithfulness ≥ 0.75 |
| Block threshold | faithfulness < 0.75 |

**On block:**
```
"I was unable to find reliable information in your documents for this query.
Please consult a licensed [legal professional / physician / relief coordinator].
This query has been logged for review."
```

**Always log:** query, retrieved chunks, final answer, faithfulness score, outcome (`delivered` or `blocked`), cited sources.

---

## 5. Modal Inference Layer

### 5.1 `/embed` endpoint

| Property | Value |
|----------|-------|
| Model | `sentence-transformers/all-MiniLM-L6-v2` |
| Hardware | CPU |
| Size | ~90MB |
| Request | `{ "texts": ["string1", "string2"] }` |
| Response | `{ "embeddings": [[0.1, 0.2, ...], ...] }` |
| Used for | Query embedding + document chunk embedding at ingestion |

---

### 5.2 `/rerank` endpoint

| Property | Value |
|----------|-------|
| Model | `cross-encoder/ms-marco-MiniLM-L-6-v2` |
| Hardware | CPU |
| Request | `{ "query": "str", "chunks": ["str", ...], "top_k": 4 }` |
| Response | `{ "ranked_chunks": [{ "text": "str", "score": 0.91 }] }` |
| Used for | Re-scoring top-k vector search results in every agent's RAG loop |

---

### 5.3 `/generate` endpoint

| Property | Value |
|----------|-------|
| Model | `mistralai/Mistral-7B-Instruct-v0.2` GGUF Q4_K_M |
| Hardware | T4 GPU |
| Request | `{ "system_prompt": "str", "context": "str", "query": "str", "stream": bool }` |
| Response | Streamed tokens or full string |
| Timeout | 60s |
| Used for | Router classification, all specialist agents, synthesis agent |

**Modal function decorators:**
```python
@app.function()                    # CPU — /embed and /rerank
@app.function(gpu="T4")           # GPU — /generate only
```

**Image build strategy:**  
Bake all model weights into the Modal image at build time using `.run_commands()` that downloads weights. No model download at request time. This eliminates model-load cold start.

---

## 6. RAG Pipeline Spec

### 6.1 Document Ingestion Pipeline

```
Input: PDF or DOCX
  │
  ├─ Security scan (injection pattern check) → reject if found
  ├─ Text extraction (PyMuPDF for PDF, python-docx for DOCX)
  ├─ Metadata tagging: { source, domain, date, language }
  ├─ Chunking: 512 tokens, 64-token overlap, sentence-boundary aware
  │           (RecursiveCharacterTextSplitter)
  ├─ Embed via Modal /embed
  └─ Store in Qdrant namespace for agent domain
```

---

### 6.2 Iterative Retrieval Loop

```python
attempt = 1
MAX_ATTEMPTS = 3
CONFIDENCE_THRESHOLD = 0.65

while attempt <= MAX_ATTEMPTS:
    # Step 1: Hybrid retrieval
    dense_chunks = qdrant_search(query_embedding, top_k=10)
    sparse_chunks = bm25_search(query_text, top_k=10)
    merged = weighted_merge(dense_chunks, sparse_chunks, w_dense=0.6, w_sparse=0.4)

    # Step 2: Rerank
    reranked = modal_rerank(query, merged, top_k=4)

    # Step 3: Score retrieval quality
    confidence = score_retrieval(query, reranked)

    if confidence >= CONFIDENCE_THRESHOLD:
        break

    # Step 4: Refine query and retry
    query = refine_query_with_llm(query, reranked)
    attempt += 1
```

**Retrieval confidence scoring:**  
Simple heuristic for v1: average of top-4 reranker scores. If average < 0.65, re-query.

---

### 6.3 Hybrid Search

| Method | Tool | Weight |
|--------|------|--------|
| Dense (semantic) | Qdrant cosine similarity | 0.6 |
| Sparse (keyword) | rank_bm25 in-memory index | 0.4 |

BM25 indexes are built in-memory per namespace at startup from stored chunk texts.

---

### 6.4 Semantic Cache

**Purpose:** Skip retrieval and generation entirely for near-duplicate queries.

```python
CACHE_SIMILARITY_THRESHOLD = 0.92

def check_cache(query_embedding, cache: dict) -> str | None:
    for cached_embedding, cached_answer in cache.items():
        if cosine_similarity(query_embedding, cached_embedding) > CACHE_SIMILARITY_THRESHOLD:
            return cached_answer
    return None
```

- Stored in-memory (resets on container restart — acceptable for v1)
- Cache structure: `{ tuple(embedding): answer_string }`
- Add Redis persistence in v2

---

## 7. Evaluation Spec

### 7.1 RAGAS Metrics

| Metric | Description | Target | Gate |
|--------|-------------|--------|------|
| `faithfulness` | Does answer contradict retrieved chunks? | ≥ 0.75 | Yes — blocks delivery |
| `answer_relevancy` | Is answer relevant to the original query? | ≥ 0.70 | No — logged only |
| `context_recall` | Did retrieval find the right chunks? | ≥ 0.65 | No — logged only |
| `context_precision` | Are all retrieved chunks actually used? | ≥ 0.60 | No — logged only |

---

### 7.2 Ground Truth Evaluation Set

- **Minimum:** 30 QA pairs
- **Sources:**
  - LegalBench (HuggingFace) — subset relevant to Bangladesh context
  - Manually written Bangladesh-specific questions from corpus documents
- **Format:**
  ```json
  {
    "question": "Can a garment worker dismissed without notice claim severance?",
    "ground_truth": "Yes. Under Bangladesh Labor Act 2006 Section 26...",
    "domain": "legal"
  }
  ```
- **Run:** `python eval/run_ragas.py` — outputs timestamped results to `eval/results/`

---

### 7.3 Audit Log

Every query logged to `audit_log.jsonl`:

```json
{
  "timestamp": "2025-01-15T10:32:00Z",
  "query_original": "...",
  "query_language": "bn",
  "query_translated": "...",
  "domains_invoked": ["legal", "health"],
  "retrieval_attempts": 2,
  "faithfulness_score": 0.82,
  "answer_relevancy": 0.79,
  "outcome": "delivered",
  "cited_sources": ["Bangladesh Labor Act 2006 §26", "DGDA Essential Medicines §4"],
  "cache_hit": false
}
```

---

## 8. Security Spec

### 8.1 Prompt Injection Detection — Document Ingestion

Scan every document chunk before storing. Reject the document if any pattern matches.

**Pattern list:**
```python
INJECTION_PATTERNS = [
    r'ignore\s+(all\s+)?previous\s+instructions',
    r'disregard\s+(the\s+)?(above|previous)',
    r'you\s+are\s+now\s+a',
    r'new\s+system\s+prompt',
    r'forget\s+(everything|all)',
    r'act\s+as\s+(if\s+you\s+are|a)',
    r'from\s+now\s+on\s+you',
]
```

If injection detected: reject document, return error `{ "error": "INJECTION_DETECTED", "chunk_index": N }`.

---

### 8.2 Input Guard — Query Time

Apply same pattern list to every incoming user query before routing.

If injection detected: return fixed response:
```
"I can only answer questions about the documents in this knowledge base."
```
Log the attempt in audit log with `outcome: "injection_blocked"`.

---

### 8.3 Document Trust Metadata

| Trust Level | Condition | Faithfulness Threshold |
|-------------|-----------|----------------------|
| `verified` | Admin explicitly confirmed document | ≥ 0.75 |
| `unverified` | Auto-ingested without confirmation | ≥ 0.85 (stricter) |

Tag stored in Qdrant chunk metadata. Trust gate applies appropriate threshold at scoring time.

---

### 8.4 PII Detection on Output (v1 — basic)

Scan generated answer for Bangladesh National ID patterns and phone number patterns before delivery:

```python
PII_PATTERNS = [
    r'\b\d{17}\b',           # Bangladesh NID
    r'\b01[3-9]\d{8}\b',     # Bangladesh mobile number
]
```

If found: strip or mask before delivery. Log occurrence.

---

## 9. Data Sources

| Agent | Document | Source | Format |
|-------|----------|--------|--------|
| Legal | Bangladesh Labor Act 2006 | bdlaws.minlaw.gov.bd | HTML → PDF |
| Legal | Prevention of Violence Against Women and Children Act 2000 | bdlaws.minlaw.gov.bd | HTML → PDF |
| Legal | NLASO Legal Aid eligibility criteria | nlaso.gov.bd | PDF |
| Legal | State Acquisition and Tenancy Act 1950 (land) | bdlaws.minlaw.gov.bd | HTML → PDF |
| Health | BRAC Shasthya Shebika CHW field manual | brac.net | PDF (request) |
| Health | DGDA Essential Medicines List | dgda.gov.bd | PDF |
| Health | Bangladesh National Health Policy 2011 | mohfw.gov.bd | PDF |
| Health | WHO Cox's Bazar Rohingya health guidelines | who.int | PDF |
| Disaster | BDRCS Cyclone Preparedness Protocols | bdrcs.org | PDF |
| Disaster | DDM Bangladesh relief distribution guidelines | ddm.gov.bd | PDF |
| Disaster | SPARRSO flood vulnerability guidelines | sparrso.gov.bd | PDF |

**Note on bdlaws.minlaw.gov.bd:** Full text of all Bangladesh Acts is available in English. Use browser's Save as PDF or a simple Python scraper (BeautifulSoup) to collect.

**Evaluation data:** LegalBench on HuggingFace (`nguyen-brat/legalBench`)

---

## 10. Tech Stack

| Layer | Technology | Version | Reason |
|-------|-----------|---------|--------|
| Web framework | FastAPI | 0.111+ | Async, native streaming support |
| LLM generation | `qwen/qwen3.8-27b` | — | High-speed, cost-effective inference for real-time Self-RAG loop |
| Inference hosting | Groq | — | Blazing fast LPUs, generous free tier |
| Embedding model | `intfloat/multilingual-e5-large` | — | Top-tier MTEB multilingual benchmark (Bengali) |
| Reranker | `BAAI/bge-reranker-v2-m3` | — | Multilingual cross-encoder improving Context Precision |
| Embed/Rerank host | Pinecone Inference API | — | Serverless endpoints avoiding free-tier compute limits |
| Vector store | Supabase (`pgvector`) | — | Consolidates relational data and vectors into one managed DB |
| RAG logic engine | Custom Python / LangGraph | — | Agentic Self-RAG loop control |
| Evaluation | Custom LLM Prompts | — | Real-time binary pass/fail grading (lower latency than Ragas) |
| Input/Output Guardrails | `openai/gpt-oss-safeguard-20b` | — | Hosted on Groq; custom policy flexibility (toxicity, injection) |
| PII Sanitization | Microsoft Presidio | — | Offline stripping of PII before chunking/embedding |
| PDF parsing | PyMuPDF (fitz) | 1.24+ | Fast, handles complex PDFs |
| Observability | LangSmith | free tier | Full pipeline tracing |
| Containerization | Docker (python:3.11-slim) | — | Render deployment |
| App hosting | Render Web Service | free tier | Dockerized FastAPI |

---

## 11. API Contracts

### POST `/query`
```json
Request:
{
  "query": "string",
  "session_id": "string (optional)"
}

Response (delivered):
{
  "answer": "string",
  "language": "bn | en",
  "citations": [
    { "source": "Bangladesh Labor Act 2006 §26", "domain": "legal" }
  ],
  "domains_used": ["legal"],
  "faithfulness_score": 0.82,
  "retrieval_attempts": 2,
  "outcome": "delivered"
}

Response (blocked):
{
  "answer": "I was unable to find reliable information...",
  "outcome": "blocked",
  "reason": "faithfulness_below_threshold",
  "faithfulness_score": 0.61
}
```

---

### POST `/ingest`
```json
Request: multipart/form-data
  file: PDF or DOCX
  domain: "legal | health | disaster"
  trust_level: "verified | unverified"

Response (success):
{
  "status": "ingested",
  "chunks_stored": 42,
  "domain": "legal",
  "injection_scan": "clean"
}

Response (rejected):
{
  "status": "rejected",
  "reason": "INJECTION_DETECTED",
  "chunk_index": 7
}
```

---

### GET `/health`
```json
Response: { "status": "ok" }
```
Returns immediately. No model calls. Used by Render health check.

---

### POST `/warmup`
Triggers Modal GPU container to pre-warm before a demo.
```json
Response: { "status": "warmed", "latency_ms": 18400 }
```

---

### GET `/eval/summary`
Returns latest RAGAS metrics from last eval run.
```json
{
  "last_run": "2025-01-15T09:00:00Z",
  "n_queries": 30,
  "faithfulness_avg": 0.81,
  "answer_relevancy_avg": 0.76,
  "context_recall_avg": 0.70,
  "block_rate": 0.13
}
```

---

## 12. Deployment Spec

### 12.1 Dockerfile (Render)

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Pre-bake translation model into image layer
# Avoids re-download on every cold start
RUN python -c "
from transformers import pipeline
pipeline('translation', model='facebook/nllb-200-distilled-600M')
"

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Target image size:** < 2GB (slim base + translation model only)

---

### 12.2 Environment Variables

| Variable | Where set | Used by |
|----------|-----------|---------|
| `MODAL_TOKEN_ID` | Render env | Modal API auth |
| `MODAL_TOKEN_SECRET` | Render env | Modal API auth |
| `QDRANT_URL` | Render env | Vector store |
| `QDRANT_API_KEY` | Render env | Vector store |
| `LANGSMITH_API_KEY` | Render env | Observability |
| `CONFIDENCE_THRESHOLD` | Render env | Retrieval gate (default: 0.65) |
| `FAITHFULNESS_THRESHOLD` | Render env | Trust gate (default: 0.75) |

---

### 12.3 Modal App Structure

```python
import modal

app = modal.App("citationguard-bd")

image = (
    modal.Image.debian_slim(python_version="3.11")
    .pip_install(["sentence-transformers", "llama-cpp-python", ...])
    .run_commands([
        # Download and cache model weights at image build time
        "python -c \"from sentence_transformers import SentenceTransformer; "
        "SentenceTransformer('all-MiniLM-L6-v2')\"",
        "python -c \"from sentence_transformers import CrossEncoder; "
        "CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')\""
    ])
)

@app.function(image=image)
@modal.web_endpoint(method="POST")
def embed(request): ...

@app.function(image=image)
@modal.web_endpoint(method="POST")
def rerank(request): ...

@app.function(image=image, gpu="T4")
@modal.web_endpoint(method="POST")
def generate(request): ...
```

---

### 12.4 Local Development Setup

For local dev: run Ollama locally, set `LLM_PROVIDER=local` to skip Modal calls.

```bash
# .env.local
LLM_PROVIDER=local          # uses Ollama locally
OLLAMA_MODEL=mistral:7b
QDRANT_URL=http://localhost:6333

# .env.production
LLM_PROVIDER=modal
MODAL_EMBED_URL=https://...
MODAL_RERANK_URL=https://...
MODAL_GENERATE_URL=https://...
```

This dual-mode setup preserves the ability to develop offline without burning Modal credits.

---

## 13. Build Order

### Phase 1 — Inference Foundation (Week 1–2)
- [ ] Modal app with `/embed`, `/rerank`, `/generate` endpoints working
- [ ] Test each endpoint independently with curl
- [ ] Basic FastAPI on Render calling Modal endpoints
- [ ] Confirm Render ↔ Modal latency is acceptable

### Phase 2 — Single-Agent RAG (Week 3–4)
- [ ] Legal agent only — ingest 3 Bangladesh law PDFs
- [ ] Qdrant Cloud connected, `legal_bd` namespace populated
- [ ] Hybrid search (dense + BM25) working
- [ ] Iterative retrieval loop (max 3 attempts)
- [ ] Manual test: 10 Bangladesh legal questions

### Phase 3 — Evaluation Pipeline (Week 5)
- [ ] RAGAS integrated — faithfulness scoring on every query
- [ ] Citation trust gate — block / deliver logic
- [ ] Audit log writing to `audit_log.jsonl`
- [ ] Ground truth eval set (30 QA pairs)
- [ ] `python eval/run_ragas.py` runs clean

### Phase 4 — Multi-Agent System (Week 6–7)
- [ ] Health + disaster agents with own Qdrant namespaces
- [ ] Router agent (classification prompt)
- [ ] Parallel agent dispatch
- [ ] Synthesis agent (merge + conflict detection)

### Phase 5 — Bengali + Security (Week 8–9)
- [ ] NLLB language handler in Render container
- [ ] Bengali query → English → Bengali response pipeline tested
- [ ] Prompt injection scanner (ingestion + query time)
- [ ] Document trust metadata on chunks
- [ ] PII scan on output

### Phase 6 — Systems Polish (Week 10)
- [ ] Semantic cache (in-memory)
- [ ] LangSmith observability traces on
- [ ] `/warmup` endpoint
- [ ] Chunking strategy experiment: test 256 / 512 / 1024 tokens, document results
- [ ] README with architecture diagram, RAGAS benchmark table, design decision log

---

## 14. Known Constraints

| Constraint | Impact | Mitigation |
|-----------|--------|-----------|
| Render 512MB RAM | Cannot run LLM | All generation via Modal |
| Modal cold start 20–40s | First GPU request is slow | `/warmup` endpoint before demos |
| Render free tier sleeps after 15min | 30–60s web cold start | Document in README; acceptable for portfolio |
| NLLB Bengali quality | Translation not perfect for formal legal text | Note limitation; suggest professional review for legal queries |
| No persistent semantic cache | Cache resets on Render restart | In-memory for v1; Redis upgrade path noted |
| RAGAS requires ground truth | Cannot evaluate without QA pairs | Build 30-pair eval set in Phase 3 before adding features |
| BM25 in-memory | Rebuilds on restart (~seconds for small corpus) | Acceptable for v1 corpus size |
| Single Qdrant free cluster | 1GB limit across all namespaces | Sufficient for initial corpus; document upgrade path |

---

## 15. Success Criteria

The project is complete when:

| Criterion | Measure |
|-----------|---------|
| Legal agent answers correctly | ≥ 80% accuracy on 30-question eval set |
| Faithfulness gate working | Block rate ≥ 10% on adversarial test queries |
| Bengali support works | 5 Bengali queries answered correctly end-to-end |
| Injection detection works | 10/10 injected documents rejected at ingestion |
| Multi-agent routing works | Cross-domain query invokes correct agent combination |
| Latency acceptable | p95 query latency < 45s including cold start |
| README complete | Architecture diagram, RAGAS table, design decisions, data sources documented |

---

*Built for CS fresher portfolio — demonstrating RAG pipeline design, multi-agent orchestration, AI evaluation, security, and production deployment on free-tier infrastructure.*
