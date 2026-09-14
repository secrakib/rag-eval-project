# Project Specification

## 1. Problem Statement

NGOs operating in Bangladesh need to answer complex queries from vulnerable clients within a single specific domain. They cannot afford commercial LLM APIs. Existing tools hallucinate without citations, operate only in English, or require expensive infrastructure.

A client needing guidance within a single domain (e.g., legal or health) needs reliable, cited answers. The system will be domain agnostic, meaning it can be configured for any single specific domain. This system provides a hosted AI assistant that:
- Runs entirely on free-tier infrastructure
- Answers in Bengali and English
- Cites every claim to a source document
- Refuses to answer when confidence falls below a safety threshold
- Serves single-domain queries effectively and reliably

## 2. Goals and Non-Goals

### Goals
- **G1:** Answer queries within a specific single domain grounded in provided documents
- **G2:** Support Bengali-language input and output
- **G3:** Block uncited or low-confidence answers before delivery
- **G4:** Run within free-tier infrastructure (e.g., Render free tier, free external inference API)
- **G5:** Pipeline is data-agnostic — any NGO can swap in their own documents
- **G6:** Every pipeline decision is documented and benchmarked

### Non-Goals
- **NG1:** Not a real-time system — latency of 15–40s is acceptable
- **NG2:** Not a general-purpose chatbot — only answers from uploaded documents
- **NG3:** Not a substitute for professionals — always recommend consultation
- **NG4:** No user authentication in v1 — single-tenant, NGO admin manages documents
- **NG5:** No fine-tuning in v1 — prompt engineering and RAG only
- **NG6:** No document versioning in v1

### Future Expansions
- **FE1:** Multi-Channel Messaging — Integrate official APIs for Telegram (100% free), Messenger (free for 24h replies), and WhatsApp Cloud API (free tier limits) to allow vulnerable clients to access the AI via familiar chat apps instead of a web interface.

## 4. Component Specifications

### 4.1 Domain Agent (Self-Reflective RAG)

**Responsibility:** Handle queries within the configured domain using an agentic Self-RAG loop. This loop actively enforces precision, recall, and faithfulness during generation.

**Optimized 3-Call Self-RAG Workflow:**
1. **Retrieve:** Fetch documents from the vector store.
2. **Input Guardrail (Call 1):** Check user query for safety, injection, and domain relevance.
3. **Combined Grade & Generate (Call 2):** A single LLM prompt evaluates if the retrieved context is sufficient. If NO, it signals for re-retrieval. If YES, it immediately generates the drafted answer.
4. **Combined Output Guardrail & Faithfulness (Call 3):** A single LLM prompt evaluates if the generated draft is completely faithful to the context AND adheres to custom safety/NGO policies.

**Fallback & Retry Logic (Max 1 Retry Per Node):**
To optimize latency and strictly protect the 60 RPM API limit, retries are capped at exactly **1 per failure point**:
- **Input Guardrail Failure:** Abort immediately (0 retries). Return: *"Sorry, I am only able to assist with questions related to [Domain]."*
- **Context Exhaustion (Retrieval Retry):** If Call 2 evaluates 'NO', the system rewrites the query and retries retrieval **once**. If the 2nd attempt fails, abort. Return: *"Sorry, we do not have sufficient information in our official documents."*
- **Faithfulness Failure (Generation Retry):** If Call 3 fails, the system retries generation **once** with a stricter grounding prompt. If the 2nd attempt fails, abort. Return: *"Sorry, I am unable to confidently provide a reliable answer based on our available documents."*

| Property | Value |
|----------|-------|
| Logic Engine | Agentic loop (e.g., custom Python logic or LangGraph) |
| Online Evaluation | Custom LLM Prompts (No external frameworks like Ragas/DeepEval to minimize latency) |
| Hardware | Render (Orchestration) + Inference Provider |

## 5. Design Decisions

### 5.1 Custom LLM Prompts vs. Evaluation Frameworks
We chose Custom LLM Prompts for real-time Self-RAG evaluation over frameworks like Ragas or DeepEval because:
1. **Lower Latency:** Frameworks execute complex, multi-step pipelines meant for offline testing. Custom prompts provide fast, binary decisions (e.g., "Pass/Fail") suitable for real-time user requests.
2. **Resource Efficiency:** Frameworks make multiple heavy LLM calls per metric, which risks exhausting free-tier compute limits. Custom prompts require only a single, optimized inference call per check.
3. **Simplicity:** Frameworks carry heavy dependencies and are heavily optimized for commercial APIs (like OpenAI). Custom prompts integrate directly and cleanly with our open-source model deployment.

## 6. Infrastructure and Inference Layer

### 6.1 Generation & Model Routing (LLM)
- **Primary Provider:** Groq
- **Model Split Strategy:** 
  - `openai/gpt-oss-safeguard-20b`: Dedicated to Input/Output Guardrails and Faithfulness grading.
  - `qwen/qwen3.8-27b`: Dedicated to Context Grading and Answer Generation.
- **Why Split Models?:** Groq rate limits are per-model (30 RPM). By splitting the 3-call workflow across two models, we effectively double our throughput to 60 RPM, allowing ~15-20 user queries per minute sustainably.
- **Mandatory Rate Limiter:** To prevent hard `429 Too Many Requests` failures, a local token-bucket rate limiter (e.g., `asyncio-throttle`) is **mandatory**. Each Groq call is wrapped and capped at 28 RPM per model.
- **In-Process Queuing:** For the MVP, if limits are hit, requests are queued in-process behind the rate limiter (holding the connection open). Since wait times at 15 queries/min are negligible (a few seconds), this avoids building complex async job-status infra.

### 6.2 Embedding
- **Provider / Host:** Pinecone Inference API
- **Model:** `multilingual-e5-large`
- **Why:** High benchmark performance on multilingual tasks (Bengali & English). Pinecone's serverless Inference API avoids hosting embedding models on free-tier compute.

### 6.3 Reranking
- **Provider / Host:** Pinecone Inference API
- **Model:** `BAAI/bge-reranker-v2-m3`
- **Why:** Multilingual cross-encoder that improves Context Precision by re-scoring retrieved chunks. Hosted serverless via Pinecone Inference API.

### 6.4 Vector Database
- **Provider:** Supabase (PostgreSQL with `pgvector`)
- **Why:** Consolidates document chunks, relational metadata, chat/evaluation logs, and vector embeddings into a single managed database with a generous free tier.

### 6.5 Document Parsing
- **Library:** `pdfplumber` (Python)
- **Why:** While slightly slower than PyMuPDF, `pdfplumber` excels at understanding document layouts, columns, and tables out-of-the-box. Since NGO reports are often heavily formatted, this ensures highly accurate text extraction without relying on heavy OCR or paid cloud APIs.

## 7. Guardrails (Security & Privacy)

### 7.1 Input Guardrail
- **Strategy:** Lightweight local Regex (first-pass) + `openai/gpt-oss-safeguard-20b` via Groq (second-pass).
- **Why:** Regex deterministically catches obvious PII (e.g., Bangladeshi NID formats, +880 phone numbers) at zero latency. The 20B Safeguard model on Groq evaluates custom policies (e.g., prompt injection, jailbreak attempts, toxicity) with negligible latency, offering greater flexibility than standard Llama Guard.

### 7.2 Chunk Sanitizer
- **Strategy:** Microsoft Presidio (Python library) running locally during offline document ingestion.
- **Why:** Using custom Regex recognizers for Bengali names, NIDs, and phone numbers, Presidio strips PII from documents *before* chunking and embedding, ensuring the vector database remains completely clean.

### 7.3 Output Guardrail
- **Strategy:** Self-RAG Evaluator (Faithfulness/Relevance) + `openai/gpt-oss-safeguard-20b` via Groq.
- **Why:** The Self-RAG loop ensures the generated answer is grounded in retrieved chunks. The final Safeguard pass enforces custom deployment policies (e.g., "no medical/legal advice beyond NGO scope", "no PII leakage") using the Harmony prompt format.
