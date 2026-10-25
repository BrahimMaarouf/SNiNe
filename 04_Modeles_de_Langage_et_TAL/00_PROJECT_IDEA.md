# DocuMind FR: a bilingual (FR/EN) document assistant, from TF-IDF to RAG to LoRA

**Field:** Language Models & NLP (Modèles de Langage et TAL) · **Duration:** 5 days (~30 h) · **Level at the end:** an AI engineer who understands NLP from tokens to production LLM systems, and can **measure** them instead of just "vibe-checking" them

---

## 1. The story (project idea)

A compliance team needs to answer questions about **AI and data regulation**: the **EU AI Act**, the **GDPR**, and Morocco's **Law 09-08**, which are available in French and English. They also receive a stream of **user feedback/reviews** that must be classified automatically.

I build **DocuMind FR** in layers. Each layer is a generation of NLP, so I understand **why** each technology exists:

| Layer | Generation | What it does in the product |
|-------|------------|------------------------------|
| 1 | Classical NLP (pre-2018) | TF-IDF + linear model to classify feedback; spaCy NER |
| 2 | Transformer encoders (2018+) | Fine-tuned French/multilingual BERT-family classifier |
| 3 | Embeddings & semantic search | Retrieve the right regulation articles for a question |
| 4 | Generative LLMs + RAG (2023+) | Answer questions **with citations**, refuse when the answer isn't in the documents |
| 5 | Evaluation & guardrails (2024–2026 practice) | Measure retrieval, faithfulness, hallucinations; resist prompt injection |
| 6 | Parameter-efficient fine-tuning | LoRA/QLoRA a small open LLM, then compare **prompting vs RAG vs fine-tuning** |

Bonus synergy: the regulatory corpus is exactly the topic of project 03 (XAI), and the RAG system becomes a **tool** for my agent in project 06.

---

## 2. Why this project covers the field

| Course concept | Milestone |
|----------------|-----------|
| Tokenisation (BPE, WordPiece, SentencePiece), normalisation, multilingual issues (French accents, Arabic/Darija) | M0 |
| Bag-of-words, TF-IDF, n-grams, linear classifiers, NER, POS | M1 |
| Word embeddings → contextual embeddings, attention, the Transformer, encoder vs decoder vs encoder–decoder, fine-tuning | M2 |
| Sentence embeddings, vector similarity, ANN indexes (HNSW), BM25, hybrid search, reranking, IR metrics | M3 |
| LLMs, decoding (temperature, top-p), prompting, context windows, RAG, structured outputs | M4 |
| LLM evaluation (LLM-as-judge, faithfulness), hallucinations, prompt injection, safety | M5 |
| PEFT: LoRA/QLoRA, SFT, quantisation, inference serving | M6 |

---

## 3. Data

| Use | Dataset |
|-----|---------|
| Classification (M1–M2) | **Allociné** French movie reviews (Hugging Face, ~200k, binary sentiment), or **MARC/Amazon reviews multilingual (French subset)** if I can find it. For a harder multi-class task, add a topic dataset. Stretch: a **Darija/Arabic** sentiment dataset (search "Moroccan Darija sentiment dataset") |
| RAG corpus (M3–M5) | **EU AI Act** (FR + EN, EUR-Lex), **GDPR** (FR + EN), Morocco **Law 09-08** (FR). Download the official HTML/PDF versions. |
| Evaluation set (M3–M5) | **I build it**: ~50 question → answer → source-article triples (30 hand-written + 20 LLM-generated **and reviewed by me**), including ~10 questions whose answer is **not** in the corpus |
| Fine-tuning (M6) | An instruction dataset I derive from my corpus/eval set (e.g. 500–1,500 examples), or a public French instruction dataset |

---

## 4. Stack (current as of October 2026)

> The model landscape changes every month. **Don't memorise model names; learn to choose.** Check the **Hugging Face Hub (trending)**, the **MTEB leaderboard** (embeddings), and current open-LLM leaderboards / arena rankings when I start.

| Purpose | Tool | Note |
|---------|------|------|
| Classical NLP | **scikit-learn**, **spaCy** (French pipeline `fr_core_news_*`) | |
| Tokenisers | Hugging Face **tokenizers** | Train my own BPE in M0 |
| Encoder models | **CamemBERT / CamemBERTa v2**, **multilingual ModernBERT-style encoders** (e.g. mmBERT), XLM-R | Choose by language + speed |
| Training | Hugging Face **transformers**, **datasets**, **evaluate**, **accelerate** | |
| Embeddings | Multilingual models from the **MTEB** top ranks, e.g. **BGE-M3**, **multilingual-E5**, **Qwen3-Embedding** family | Must handle French |
| Vector store | **Qdrant** (Docker) or **pgvector** (Postgres) | LanceDB/Chroma are fine for prototypes |
| Lexical search | **BM25** (rank-bm25, or the hybrid search built into Qdrant/Postgres) | |
| Reranker | A cross-encoder reranker (e.g. the BGE-reranker family) | |
| LLM runtime (local) | **Ollama** or **llama.cpp**; **vLLM** for GPU serving | A small open instruct model in the **1–8B** range (recent Qwen, Gemma, Mistral, Llama, or Phi families; check the licences) |
| LLM (API, optional, **free**) | **GitHub Models** or **Google AI Studio** free tier (no card) | Compare against local; send only public text |
| Orchestration | **Plain Python first**, then optionally LlamaIndex / LangChain / Haystack | I must understand the raw pipeline before a framework hides it |
| Structured output | **Pydantic** + JSON-schema-constrained generation | |
| Evaluation | **RAGAS** or **DeepEval**, plus my own metrics; **MLflow 3** GenAI tracing/evaluation or **Langfuse** for traces | |
| Fine-tuning | **PEFT**, **TRL** (SFTTrainer), **Unsloth** (fast QLoRA on a single GPU), **bitsandbytes** | On **my RTX 4080 (12 GB)**; Colab/Kaggle free GPUs as a backup |
| Serving & UI | **FastAPI** (+ streaming), **Gradio/Streamlit**, Docker | |

---

## 5. Architecture

```
 Feedback text ──▶ [TF-IDF + LogReg] vs [fine-tuned CamemBERT] ──▶ label + confidence

 Regulation PDFs/HTML (FR/EN)
        │ parse → clean → chunk by article/paragraph (+ metadata: law, article, lang)
        ▼
  ┌──────────────┐  dense embeddings   ┌──────────┐
  │ chunk store  │────────────────────▶│ Qdrant / │
  └──────────────┘  BM25 index         │ pgvector │
        question ─▶ hybrid retrieve (top-30) ─▶ rerank (top-5) ─▶ prompt builder
                                                              │ (system rules, citations,
                                                              ▼  "say I don't know")
                                                     LLM (local / API)
                                                              │ structured answer + sources
                                                              ▼
                                  FastAPI ─▶ UI      + traces ─▶ MLflow / Langfuse
                                                     + eval suite (RAGAS/DeepEval + golden set)
 LoRA track:  base small LLM ─▶ QLoRA SFT ─▶ adapter ─▶ compare: prompt-only vs RAG vs FT vs FT+RAG
```

---

## 6. Final deliverables
1. Repo `documind/` with `src/` (classic, encoder, retrieval, rag, eval, finetune), `data/` (scripts, not raw data), `eval/` (golden set + reports), `api/`, `docs/`.
2. A **classification benchmark** table: TF-IDF vs fine-tuned encoder (accuracy, macro-F1, latency, model size).
3. A **retrieval benchmark**: BM25 vs dense vs hybrid vs hybrid + rerank (Recall@5, MRR, nDCG@10).
4. A **RAG evaluation report**: faithfulness, answer relevance, context precision, correct-refusal rate, latency, **estimated** cost per query (at public API prices; I spend $0).
5. A **fine-tuning report**: prompt vs RAG vs LoRA vs LoRA + RAG on the same eval set.
6. A Dockerised API + a simple UI demo (and a short screen recording).

## 7. If I skipped earlier projects
None are required. Docker (project 01) helps for Qdrant and the API. My **RTX 4080 (12 GB)** handles M2 and M6 locally; Colab/Kaggle free GPUs are a backup. The whole project costs **$0 and needs no card**: see the zero-cost rule at the top of `01_MILESTONES.md`.
