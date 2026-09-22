# Financial RAG Pipeline — Nova Tech Solutions

A Retrieval-Augmented Generation (RAG) pipeline built in n8n that answers questions
over a small set of financial documents (a 10-K excerpt, an earnings call transcript,
and a press release) for a fictional company, Nova Tech Solutions, Inc.

This repo accompanies the course deliverable: a working RAG pipeline, a comparison
of three chunking strategies, and a reranking impact analysis. See
`Financial_RAG_Report.docx` for the full write-up.

## Contents

| File | Description |
|---|---|
| `Financial_RAG_Report.docx` | Full report: methodology, results, limitations, deviations from the course guide |
| `documents/nova_tech_10k_extended.txt` | 10-K excerpt (source document) |
| `documents/nova_tech_earnings_transcript.txt` | Earnings call transcript (source document) |
| `documents/nova_tech_press_release.txt` | Press release (source document) |
| `workflows/Ingestion_workflow.json` | Fetches documents, splits 3 ways (fixed / large-chunk / section-based), embeds, writes to Pinecone |
| `workflows/QA_workflow.json` | Chat-triggered AI Agent with two retrieval tools, Groq LLM, conversation memory |
| `workflows/Evaluation_workflow.json` | Runs 5 test questions through the Q&A workflow and scores pass/fail |
| `workflows/Retrieval_Benchmark_workflow.json` | Isolated retrieval comparison: baseline vs. reranked, across all 3 chunking strategies |
| `screenshots/` | Supporting evidence: Pinecone counts, benchmark output, evaluation report, agent tool-routing logs |

## Architecture

```
Ingestion:  GitHub (raw .txt) → HTTP Request → Set Content
              ├─ Fixed-500 splitter    → HF embeddings → Pinecone (strategy=fixed)
              ├─ Large-1500 splitter   → HF embeddings → Pinecone (strategy=semantic)
              └─ Section splitter      → HF embeddings → Pinecone (strategy=section)

Q&A:        Chat Trigger → AI Agent (Groq gpt-oss-120b)
              ├─ search_fixed_chunks    (Pinecone, filter: strategy=fixed)
              ├─ search_semantic_chunks (Pinecone, filter: strategy=semantic)
              └─ Simple Memory (windowed, session-keyed)

Evaluation: Manual Trigger → Test Cases → Execute Sub-workflow (Q&A)
              → Simple Scoring → Aggregate → Generate Report

Benchmark:  Manual Trigger → Test Queries (10 × 3 strategies = 30 items)
              ├─ Retrieve baseline (top-3, no rerank) → Tag → Score
              └─ Retrieve rerank (top-8 → Cohere rerank-v3.5 → top-3) → Tag → Score
              → Merge → Summary Table
```

## Setup

### Accounts and credentials needed
- **n8n** (Cloud or self-hosted) with the LangChain/AI nodes available.
- **Pinecone**: one index, dimension `384`, metric `cosine` (matches the Hugging Face
  `gte-small` embedding model used throughout).
- **Hugging Face**: an Inference API token with the **"Make calls to Inference
  Providers"** permission enabled (a plain Read-only token is not sufficient).
- **Groq**: an API key. This project uses `openai/gpt-oss-120b` (Groq's recommended
  replacement for the now-retired `llama-3.3-70b-versatile`). The free tier's rate
  limits (8,000 tokens/min) are too low for multi-tool agent questions; the
  Developer (pay-as-you-go) tier was used here and cost well under $1 for the
  entire project.
- **Cohere**: an API key for the Reranker Cohere node (`rerank-v3.5`). The free
  trial key is limited to 10 calls/minute, which is too slow for a 30-query
  benchmark run in one execution — a production key or a batched/throttled
  workflow (Loop Over Items + Wait) is recommended.
- **GitHub**: the three source documents must be hosted somewhere n8n's HTTP
  Request node can fetch as raw text (this project used a public GitHub repo's
  raw file URLs).

### Import order
1. Create the Pinecone index first (dimension 384, metric cosine).
2. Import and run `Ingestion_workflow.json` once. It writes ~39 chunks total
   across the three strategies. **Do not run it twice** without first clearing
   the index — it will duplicate every record.
3. Import `QA_workflow.json`. Update the Pinecone credential, Hugging Face
   credential, and Groq credential to your own. Test with a chat message before
   proceeding.
4. Import `Evaluation_workflow.json`. It calls the Q&A workflow by name/ID via
   Execute Sub-workflow — update that reference to point at your imported copy.
5. Import `Retrieval_Benchmark_workflow.json`. Add your Cohere credential to the
   Reranker Cohere node.

### Verifying ingestion
After running the Ingestion workflow, check the Pinecone console:
- `strategy == fixed` → ~20 records
- `strategy == semantic` → ~6 records
- `strategy == section` → ~13 records
- Total record count → ~39

## Known limitations
See Section 11 of the full report. In short: this is a small corpus (3 documents),
so retrieval metrics move by large increments per query and should be read as
directionally suggestive rather than statistically robust. Full discussion,
including every deviation made from the original course template and why, is in
Section 12 of the report.

## Author's note on the "semantic" chunking strategy
The course template's "semantic" strategy is a larger fixed-size chunk (1,500 vs.
500 characters), not a meaning-based split. This project adds a genuine
structure-based ("section") strategy alongside it and is transparent about this
distinction throughout the report — see Section 4 (Methodology).
