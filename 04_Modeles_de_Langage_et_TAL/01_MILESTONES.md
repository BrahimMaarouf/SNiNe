# DocuMind FR: milestones

> Format: **Goal → Why now → Concepts → What to build (Core / Stretch) → Best practices → Definition of Done → Pitfalls**. Total: 7 milestones over about 5 days.

---

## 💸 Zero-cost rule: this whole project costs $0 and needs no card

My laptop has an **NVIDIA RTX 4080 Laptop GPU with 12 GB VRAM**. That's enough to fine-tune the encoder (M2), run local LLMs (M4–M5), and do QLoRA on a 1–4B model (M6), all **for free**.

| Need | Free choice ✅ | Avoid ❌ |
|------|---------------|---------|
| Data | Allociné (Hugging Face), EU AI Act / GDPR (EUR-Lex), Law 09-08 (official public texts) | Paid datasets |
| GPU | **My RTX 4080 (12 GB)**, with the CUDA build of PyTorch installed via uv. Backup: **Colab free** / **Kaggle free GPU** (phone verification, no card) | Colab Pro, paid cloud GPUs |
| LLM (main) | **Ollama** locally. A 7–8B instruct model in 4-bit (~5 GB) runs fast on 12 GB; even ~14B in 4-bit fits | Paid APIs |
| LLM (optional API comparison) | Free tiers that need **no card**: **GitHub Models** (free, rate-limited, works with my GitHub account) or **Google AI Studio** (Gemini API free tier). I check their current limits first | Any API that asks for a card or credits |
| Embeddings + reranker | Open models from Hugging Face, run locally | Paid embedding APIs |
| Vector DB | **Qdrant** or **pgvector** in Docker (open source) | Managed vector DB paid tiers |
| Tracking / tracing | **MLflow** locally or **self-hosted Langfuse** (Docker) | Weights & Biases / LangSmith paid tiers |
| Evaluation | **RAGAS** / **DeepEval** (open source) with my local model (or the free API) as the judge | Paid eval platforms |
| Sharing the model | **Hugging Face Hub** (free) | — |

**My hard rules**
- Never enter a card. If a provider asks for billing details, I don't use it.
- **Free API tiers may use my prompts to improve their models**, so I only send public regulation text and never personal data or secrets. That's real-world data-governance practice.
- "Cost" metrics are **estimates**: tokens counted × the provider's public price per 1M tokens. I measure what it *would* cost a company without spending anything, and for local models I report tokens/second and latency.

---

## M0: Text is just numbers: tokenisation and foundations (Day 1 morning, ~3 h)

**Goal:** understand how raw text becomes model input, and why that matters for French, Arabic, and Darija.

**Why now:** every NLP and LLM behaviour (cost, context length, multilingual quality, weird failures with numbers and spelling) starts at the **tokeniser**. Engineers who skip this can't explain why an Arabic prompt costs 3× more tokens than an English one.

**Concepts to learn**
- Normalisation (Unicode NFC/NFKC, accents, case), sentence splitting, word vs subword vs character tokens
- **BPE, WordPiece, Unigram/SentencePiece**, byte-level BPE
- Vocabulary size trade-offs; **token fertility** (tokens per word) across languages
- Special tokens, padding, truncation, attention masks
- A short history: n-grams → word2vec/GloVe → ELMo → Transformers (BERT/GPT) → instruction-tuned LLMs

**What to build**
- Core: train a small **BPE tokeniser** on the French regulatory corpus with HF `tokenizers` (vocab 8k–16k).
- Core: compare **my tokeniser vs 3 pretrained ones** (a French encoder, a multilingual encoder, a recent LLM tokeniser) on the same French, English, and Arabic/Darija sentences. Measure **fertility** and record it in a table.
- Core: write `docs/nlp_timeline.md`: one paragraph per NLP generation, saying what problem it solved.

**Definition of Done**
- [ ] Tokeniser trained and the comparison table done
- [ ] I can explain how BPE builds its vocabulary step by step

---

## M1: Classical NLP baseline (Day 1 afternoon, ~3 h)

**Goal:** build a strong, fast, cheap baseline classifier, and a small NER/information-extraction step.

**Why now:** in industry I **always** start with a baseline. TF-IDF + logistic regression often gets within a few points of a Transformer at 1/1000 of the cost, and without a baseline I can't justify a bigger model.

**Concepts to learn**
- Bag-of-words, TF-IDF, n-grams (word + char), stop words (and why I often should *not* remove them for sentiment)
- Linear models for text (logistic regression, linear SVM), class imbalance
- Metrics: accuracy vs **macro-F1**, confusion matrix, **error analysis**
- spaCy pipelines: tokenisation, POS, lemmatisation, **NER** in French

**What to build**
- Core: a TF-IDF (word 1–2 grams + char 3–5 grams) + LogReg classifier on Allociné with a proper train/val/test split. Record macro-F1, latency per 1k texts, and model size.
- Core: **error analysis**: read 30 misclassified examples and categorise them (sarcasm, negation, mixed sentiment, label noise…). This is a skill that many engineers neglect.
- Core: use spaCy French NER to extract organisations, dates, and laws from the regulation corpus, and count the most frequent entities.
- Stretch: try the same pipeline on a Darija/Arabic dataset and note what breaks.

**Definition of Done**
- [ ] Baseline metrics + latency + size recorded
- [ ] Error analysis categories written
- [ ] NER extraction working

---

## M2: Transformers: fine-tune an encoder (Day 2, ~5 h)

**Goal:** understand the Transformer and fine-tune a French/multilingual encoder that beats my baseline, then **quantify whether it's worth it**.

**Why now:** I now have a baseline to beat. Encoders (BERT family) are still the workhorses of production NLP for classification, NER, and embeddings, and they're cheaper and more controllable than LLMs for these tasks.

**Concepts to learn**
- Self-attention (Q, K, V), multi-head attention, positional encodings (absolute, RoPE), layer norm, residuals
- Encoder-only (BERT) vs decoder-only (GPT) vs encoder–decoder (T5); masked LM vs causal LM objectives
- Transfer learning, fine-tuning heads, learning-rate schedules (warmup), early stopping, overfitting
- Modern encoders (ModernBERT-style: long context, faster attention) and French ones (CamemBERT/CamemBERTa v2)
- Read: "The Illustrated Transformer" (Jay Alammar) and "Attention Is All You Need" (sections 3.1–3.3)

**What to build**
- Core: fine-tune a French or multilingual encoder on Allociné (a subset of ~20–40k examples is enough) with HF `Trainer` on **my RTX 4080** (use fp16/bf16; Colab/Kaggle free GPUs are a backup). Log runs to a local **MLflow**.
- Core: **benchmark table**: baseline vs encoder covering macro-F1, latency (CPU *and* GPU), model size, and training cost (time). Write a **recommendation**: which model would I deploy, and why?
- Core: re-run the M1 error analysis. Which error categories did the Transformer fix, and which remain?
- Stretch: export to ONNX or apply dynamic quantisation and measure the CPU speed-up.
- Stretch: push the model to the Hugging Face Hub with a **model card**.

**Definition of Done**
- [ ] Fine-tuned model beats the baseline (or I explain why not)
- [ ] Benchmark + deployment recommendation written
- [ ] I can draw a Transformer block and explain the attention formula

**Pitfalls:** evaluating on the validation set I tuned on; using a sequence length that's too long (slow) or too short (truncated reviews).

---

## M3: Embeddings, semantic search, and retrieval evaluation (Day 3, ~5 h)

**Goal:** build and **measure** a retrieval system over the regulation corpus. This is the R in RAG.

**Why now:** in real RAG systems, **most quality problems are retrieval problems**. If the right passage isn't retrieved, no LLM can answer correctly. Build and evaluate retrieval **before** adding an LLM so I can tell which component is failing.

**Concepts to learn**
- Sentence embeddings (bi-encoders, contrastive training), cosine similarity, normalisation
- **Chunking strategies**: fixed-size + overlap vs structure-aware (by article/paragraph) vs semantic; metadata (law, article number, language)
- Vector indexes: flat vs **HNSW** vs IVF; recall/latency trade-offs
- **BM25** (lexical) vs dense; **hybrid search** and **Reciprocal Rank Fusion (RRF)**
- **Cross-encoder reranking**
- Cross-lingual retrieval (an English question → French passage)
- IR metrics: **Recall@k, MRR, nDCG@k**
- The **MTEB leaderboard**: how to choose an embedding model

**What to build**
- Core: a document parser: PDF/HTML → clean text, **chunked by article** with metadata. Inspect the chunks manually (garbage in, garbage out).
- Core: **build the golden evaluation set** (~50 questions with the expected article IDs, including ~10 unanswerable ones). This dataset is my most valuable asset; real teams guard theirs carefully.
- Core: index the chunks in Qdrant (or pgvector) with a multilingual embedding model, and set up a BM25 index as well.
- Core: **retrieval benchmark**: BM25 vs dense vs hybrid (RRF) vs hybrid + reranker, reporting Recall@5, MRR, nDCG@10, and latency. Test **two chunking strategies**.
- Stretch: test cross-lingual retrieval (EN questions over FR docs).

**Definition of Done**
- [ ] Golden set created and versioned
- [ ] Retrieval benchmark table with ≥ 4 configurations × 2 chunking strategies
- [ ] The best configuration is chosen and justified

---

## M4: RAG: grounded generation with an LLM (Day 4 morning, ~4 h)

**Goal:** generate correct, **cited** answers, and **refuse** when the documents don't contain the answer.

**Why now:** retrieval is now measured and reliable, so any remaining failure is a generation problem, and I can debug it in isolation.

**Concepts to learn**
- How LLMs generate: next-token prediction, **decoding** (greedy, temperature, top-p), context window limits, "lost in the middle"
- Instruction-tuned vs base models; chat templates; system prompts
- Prompt engineering for RAG: grounding instructions, citation format, refusal policy, few-shot examples
- **Structured outputs** (JSON schema / Pydantic) for answer + citations + confidence
- Local inference: quantisation formats (GGUF, 4-bit/8-bit), Ollama/llama.cpp; GPU serving with vLLM (continuous batching, PagedAttention, at the concept level)
- Streaming responses

**What to build**
- Core: a RAG pipeline in **plain Python**: question → hybrid retrieve → rerank → prompt builder → LLM (local through Ollama, optionally a **free-tier API model** such as GitHub Models or Google AI Studio for comparison, with no card) → **structured answer** `{answer, citations[article, law], answerable: bool}`.
- Core: a **refusal behaviour**: when retrieval scores are low or the model says it's unanswerable, reply "not found in the provided documents".
- Core: a FastAPI endpoint with streaming + a minimal Gradio/Streamlit UI.
- Core: **trace every request** (retrieved chunks, prompt, output, tokens, latency) with MLflow tracing or Langfuse.
- Stretch: rebuild the same pipeline with LlamaIndex or LangChain and list what the framework hides from me.

**Definition of Done**
- [ ] Answers include correct article citations on the golden set (I check them manually first)
- [ ] Unanswerable questions are refused
- [ ] API + UI + traces working

---

## M5: Evaluation, hallucinations, and guardrails (Day 4 afternoon, ~3 h)

**Goal:** **measure** my RAG system the way production teams do, and harden it against attacks.

**Why now:** a demo that "looks good" is not a product. In 2026, **evaluation-driven development** (change something → re-run the eval → compare) is the standard way LLM apps are built. It comes after M4 because I need a working system to evaluate.

**Concepts to learn**
- RAG metrics: **faithfulness/groundedness**, answer relevance, context precision/recall, correct-refusal rate
- **LLM-as-judge**: its biases (position, verbosity, self-preference) and **calibrating the judge** against my own human labels
- Hallucination types (intrinsic vs extrinsic)
- **Prompt injection** (direct and *indirect*, through retrieved documents), jailbreaks, data leakage; OWASP **Top 10 for LLM Applications**
- Cost and latency budgets (tokens × price, p95 latency)

**What to build**
- Core: an **automated eval suite** (RAGAS or DeepEval + my own metrics) over the golden set, producing a report. Hand-label 20 answers myself and check how well the LLM-judge agrees with me.
- Core: **regression testing**: change one thing (chunk size, top-k, prompt, model), re-run, and compare. Keep a results table in `eval/history.md`.
- Core: **indirect prompt injection test**: plant a malicious instruction inside a document chunk ("ignore previous instructions and…"), then measure and mitigate (delimiters, instruction hierarchy, output validation, refusing to follow instructions found in context).
- Core: report the **estimated** cost per query (tokens × public API price, so $0 spent) and p95 latency (local vs free API).
- Stretch: run the eval suite in CI (it connects to project 05).

**Definition of Done**
- [ ] Eval report with all metrics + judge–human agreement
- [ ] At least 3 experiments recorded in the history table
- [ ] Prompt-injection test done, with a mitigation and a before/after comparison

---

## M6: Parameter-efficient fine-tuning (LoRA/QLoRA) and packaging (Day 5, ~6 h)

**Goal:** fine-tune a small open LLM with QLoRA and decide **with evidence** when fine-tuning beats prompting or RAG.

**Why now:** this is the last milestone because fine-tuning is the **most expensive** option and should only be used once I've shown that prompting and RAG aren't enough. A key real-world lesson: **RAG injects knowledge; fine-tuning shapes behaviour, format, and style.**

**Concepts to learn**
- Pre-training vs SFT vs preference tuning (RLHF, **DPO**), at the concept level
- **LoRA** (low-rank adapters: rank r, alpha, target modules) and **QLoRA** (4-bit NF4 base + LoRA)
- Chat templates, loss masking on the prompt, overfitting on small datasets, catastrophic forgetting
- Merging adapters, quantising for inference (GGUF), serving adapters (vLLM multi-LoRA, at the concept level)

**What to build**
- Core: create a training set of **500–1,500 examples** (instruction → ideal answer in my strict citation/JSON format) from my corpus. Hold out the golden set; **never train on it**.
- Core: QLoRA-fine-tune a **small** instruct model (≈1–4B) with TRL/Unsloth on **my RTX 4080 (12 GB)**: 4-bit base, a small batch + gradient accumulation, gradient checkpointing. Colab/Kaggle free GPUs are a backup. Log the training and validation loss.
- Core: **the decisive experiment**: compare (a) base + prompt, (b) base + RAG, (c) LoRA, (d) LoRA + RAG on my eval suite (correctness, format validity, faithfulness, refusal rate, latency). Write the conclusion.
- Core: package everything: Dockerised API (classifier + RAG endpoints), `.env.example`, README with the architecture diagram, all benchmark tables, and **known limitations**.
- Stretch: export the merged model to GGUF and serve it through Ollama.
- Stretch: try a small **DPO** run on 100 preference pairs (good vs bad answers).

**Definition of Done**
- [ ] Adapter trained and loss curves saved
- [ ] 4-way comparison table + a written conclusion
- [ ] Docker image runs the full demo
- [ ] README complete

**Pitfalls:** training on the evaluation data (contamination); too few or low-quality examples; expecting fine-tuning to add facts reliably.

---

## Self-check: interview questions

1. Why does the same sentence cost more tokens in Arabic than in English? How does BPE work?
2. When would I deploy TF-IDF + LogReg instead of a Transformer?
3. Explain self-attention with Q, K, V. Why divide by √d?
4. Encoder vs decoder models: which would I use for classification, for embeddings, and for chat, and why?
5. My RAG system gives wrong answers. How do I find out whether retrieval or generation is at fault?
6. BM25 vs dense retrieval: when does each one fail? What does RRF do?
7. What is a cross-encoder reranker, and why not use it for the first-stage retrieval?
8. How do I evaluate an LLM answer automatically, and what are the risks of LLM-as-judge?
9. What is indirect prompt injection? Give 3 mitigations.
10. LoRA: what is actually trained? What do the rank and alpha control? Why does QLoRA fit on a small GPU?
11. RAG vs fine-tuning: give a case where each one is clearly the right choice.

## References to search
- **"Speech and Language Processing"** (Jurafsky & Martin, 3rd ed. draft, free online)
- Hugging Face **LLM Course** (free) and the **smol course** (fine-tuning)
- "The Illustrated Transformer" (Jay Alammar); Andrej Karpathy, "Let's build GPT" and "Let's build the GPT Tokenizer"
- **MTEB leaderboard**; Qdrant docs on hybrid search
- RAGAS / DeepEval docs; **OWASP Top 10 for LLM Applications** (latest edition)
- LoRA paper (Hu et al. 2021), QLoRA paper (Dettmers et al. 2023), the Unsloth docs
- Chip Huyen, **"AI Engineering"** (O'Reilly, 2025), the reference book for production LLM systems
