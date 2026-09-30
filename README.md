# PDF RAG Chat — Showcase

A chatbot that answers questions **only from your PDFs**, with every answer cited to the **file and page** it came from.
If the documents don't contain the answer, it says so instead of guessing.

> This repo is a showcase (screenshots + behaviour). The source code is private and available on request / as part of a project.

![Chat demo](screenshots/demo.gif)

## What it does

- Upload a handful of PDFs (policies, manuals, contracts, reports)
- Ask questions in plain language
- Get concise answers with `[n]` citations → expand to see **file, page and the exact passage**
- Out-of-scope questions are refused, not hallucinated
- Partly answerable questions: answers the covered part, states what's missing
- Broken files are rejected; scanned PDFs without a text layer are flagged as not searchable
- Click any citation to open the PDF at the cited page
- Ingest cleans up real-world PDFs: skips tables of contents, reference lists and duplicate files (`handbook.pdf` + `handbook_FINAL.pdf`)

## Tested on 1,000 pages of real documents

The system was benchmarked on 12 long public documents with 48 hand-verified questions. Chunking and
retrieval settings were changed one at a time, and each change was measured.

**83% answer accuracy · 93% cite the right document · 8/8 unanswerable questions refused, with no invented facts**

| Corpus | Documents | Pages | Answer accuracy |
|---|---|---|---|
| Employee handbooks | Indiana, Nevada, Tennessee state handbooks | 143 | 90% |
| SEC 10-K filings | Apple, NVIDIA, Microsoft | 333 | 90% |
| Clinical trial papers | PubMed Central Open Access (2 near-identical heart-failure trials + 1 PTSD trial) | 42 | 80% |
| Novels | Project Gutenberg: Pride and Prejudice, Sherlock Holmes, Frankenstein | 476 | 70% |

**When it's wrong, it's wrong safely:** none of the misses is a made-up answer. Each one is
"not found" or a partial answer that says which part the documents don't cover.

What was measured (answer accuracy over the 40 answerable questions):

| Setting | Accuracy | Prompt tokens / question |
|---|---|---|
| 256-token chunks | 60% | ~1,100 |
| 1024-token chunks | 65% | ~3,600 |
| 512-token chunks | 73% | ~2,100 |
| 512 + reference-list filtering | 75% | ~2,100 |
| **512 + neighbouring-chunk expansion (default)** | **83%** | ~4,900 (< $0.001 / question) |

Takeaways: mid-size chunks beat both small and large ones. The biggest single gain came from pulling in the
chunks next to each hit, which fixes tables split across chunks and answers sitting just after the matched
passage. That gain isn't just "more context": 1024-token chunks without expansion scored lower.

## Real example outputs

Sample corpus: 3 fictional company documents (employee handbook, travel policy, product manual).

| Question | Answer | Cited source |
|---|---|---|
| How many days of PTO can I carry over, and when do they expire? | Up to 5 unused days may be carried over into the next calendar year [1]. Carried-over days expire on March 31 [1]. | `acme_employee_handbook.pdf`, p.2 |
| How do I factory reset the X200, and is water damage covered? | Hold power + volume-down for 12 seconds until the LED blinks orange three times [2]. Water damage is not covered by the 2-year limited warranty [1]. | `widgetpro_x200_manual.pdf`, p.2 + p.3 |
| What is the international per diem, and who is the CEO? | The daily meal allowance for international travel is USD 100 [1]. The documents don't contain information about the CEO. | `acme_travel_policy.pdf`, p.2 |
| What is the capital of France? | I couldn't find the answer in the provided documents. | — (correct refusal) |

## Screenshots

| Chat with citations | Refusal on out-of-scope question |
|---|---|
| ![](screenshots/chat-citations.png) | ![](screenshots/refusal.png) |

## How it works

```
Ingest:  PDFs ─▶ text per page ─▶ drop TOC / duplicate pages ─▶ 512-token chunks (page kept)
                ─▶ drop reference lists ─▶ embeddings ─▶ vector index + BM25 keyword index

Chat:    question ─▶ hybrid search (vector + BM25) ─▶ top-5 + neighbouring chunks ─▶ grounded prompt ─▶ LLM
                   ─▶ answer with [n] markers ─▶ only cited passages returned as sources
                   ─▶ no valid citation? → treated as ungrounded → refusal
```

**Stack:** Python · FastAPI · LangChain · FAISS · OpenAI API (or any OpenAI-compatible model, e.g. DeepSeek; local embeddings via Ollama) · Streamlit

**Engineering:** REST API with OpenAPI docs, pinned dependencies, 20 offline unit tests (fake LLM + fake embeddings), config via `.env`, and a reusable benchmark harness (retrieval hit@k, LLM-judged answer accuracy, refusal checks) to tune on each client's own documents.

## Extensible to

Qdrant / Pinecone / Supabase Vector · Next.js frontend · OCR for scanned PDFs · Slack / web-widget delivery · multi-turn conversation · reranking
(see also: [healthcare-agentic-rag](https://github.com/PatrickSun93/healthcare-agentic-rag) — hybrid retrieval, reranking, LangGraph agent, eval harness)

## Contact

Available for RAG / document-AI projects — reach out via my freelance profile.
