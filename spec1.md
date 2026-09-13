# Project Specification

## 1. Problem Statement

NGOs operating in Bangladesh need to answer complex queries from vulnerable clients within a single specific domain. They cannot afford commercial LLM APIs. Existing tools hallucinate without citations, operate only in English, or require expensive infrastructure.

A client needing guidance within a single domain (e.g., legal or health) needs reliable, cited answers. The system will be domain agnostic, meaning it can be configured for any single specific domain. This system provides a hosted AI assistant that:
- Runs entirely on free-tier infrastructure (Modal + Render)
- Answers in Bengali and English
- Cites every claim to a source document
- Refuses to answer when confidence falls below a safety threshold
- Serves single-domain queries effectively and reliably

## 2. Goals and Non-Goals

### Goals
- **G1:** Answer queries within a specific single domain grounded in provided documents
- **G2:** Support Bengali-language input and output
- **G3:** Block uncited or low-confidence answers before delivery
- **G4:** Run within Modal free tier ($30/month) and Render free tier (512MB RAM)
- **G5:** Pipeline is data-agnostic — any NGO can swap in their own documents
- **G6:** Every pipeline decision is documented and benchmarked

### Non-Goals
- **NG1:** Not a real-time system — latency of 15–40s is acceptable
- **NG2:** Not a general-purpose chatbot — only answers from uploaded documents
- **NG3:** Not a substitute for professionals — always recommend consultation
- **NG4:** No user authentication in v1 — single-tenant, NGO admin manages documents
- **NG5:** No fine-tuning in v1 — prompt engineering and RAG only
- **NG6:** No document versioning in v1

## 4. Component Specifications

### 4.1 Domain Agent (Self-Reflective RAG)

**Responsibility:** Handle queries within the configured domain using an agentic Self-RAG loop. This loop actively enforces precision, recall, and faithfulness during generation.

**Self-RAG Loop Workflow:**
1. **Retrieve:** Fetch documents from the vector store.
2. **Grade Context (Enforcing Context Precision):** A custom LLM prompt evaluates each retrieved chunk. Irrelevant chunks are discarded to remove noise.
3. **Assess Sufficiency (Enforcing Context Recall):** A custom LLM prompt checks if the remaining chunks contain *all* necessary information. If missing, it rewrites the search query and retrieves again.
4. **Generate:** Draft an answer using the highly precise and sufficient context.
5. **Grade Faithfulness & Relevance:** A custom LLM prompt critiques the draft. If it hallucinated (failed faithfulness) or missed the point (failed relevance), it rewrites the answer.

| Property | Value |
|----------|-------|
| Logic Engine | Agentic loop (e.g., custom Python logic or LangGraph) |
| Online Evaluation | Custom LLM Prompts (No external frameworks like Ragas/DeepEval to minimize latency) |
| Hardware | Render (Orchestration) + Modal (Inference) |
