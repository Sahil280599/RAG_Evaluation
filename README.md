# RAG & chatbot evaluation with LangSmith

This repo is a **single Jupyter notebook** that shows how to **evaluate LLM apps with [LangSmith](https://smith.langchain.com)**: first a plain chatbot, then a **retrieval-augmented** bot with richer metrics.

---

## What’s in the notebook

| Part | What it does |
|------|----------------|
| **Environment** | Loads API keys, enables LangSmith tracing. |
| **Chatbot dataset** | Creates or reuses a LangSmith dataset (`MychatBot`) with Q&A gold answers. |
| **Simple evaluators** | `correctness` (LLM judge) and `concision` (length vs reference). |
| **Simple app** | `my_app` + `ls_target` → `client.evaluate` on `MychatBot`. |
| **RAG pipeline** | Load web pages → chunk → embed → `InMemoryVectorStore` → retriever. |
| **`rag_bot`** | Retrieves docs, calls `gpt-4o-mini`, returns `{"answer", "documents"}`. |
| **RAG dataset** | Separate dataset (`rag_dataset_name`, e.g. `"Sahil Dataset name"`) with agent-themed Q&A. |
| **RAG evaluators** | Correctness vs reference, relevance to question, groundedness vs retrieved text, retrieval relevance. |
| **RAG evaluation** | `rag_target` + `client.evaluate(..., data=rag_dataset_name, ...)`. |

Run sections **in order** the first time: later cells depend on `client`, `retriever`, `llm`, `rag_bot`, and evaluator definitions.

---

## Requirements

- **Python 3.10+** (3.13 is fine)
- **[LangSmith](https://smith.langchain.com)** account and API key
- **[OpenAI](https://platform.openai.com/)** API key (chat + embeddings)

Install dependencies:

```bash
pip install -r requirements.txt
```

Or manually: `python-dotenv`, `langsmith`, `openai`, `jupyter`, `langchain-core`, `langchain-community`, `langchain-openai`, `langchain-text-splitters`, and **`beautifulsoup4`** (needed for `WebBaseLoader` HTML parsing).

---

## Configuration (`.env`)

Create `.env` in the project root. **Do not commit real keys.**

| Variable | Purpose |
|----------|---------|
| `OPENAI_API_KEY` | OpenAI SDK (preferred). |
| `OPEN_API_KEY` | Optional alias; the notebook maps it to `OPENAI_API_KEY` if the standard name is empty. |
| `LANGSMITH_API_KEY` | LangSmith API access. |

The notebook sets `LANGSMITH_TRACING=true` so runs and LLM calls can appear in LangSmith.

---

## Notebook map (high level)

1. **Cell 0** — `load_dotenv`, set `OPENAI_API_KEY` and LangSmith env vars.  
2. **Cells 1–6** — Chatbot baseline: dataset **get-or-create** (avoids 409 if the name already exists), evaluators, `my_app`, `ls_target`, `evaluate` on `dataset_name` (`MychatBot`).  
3. **Cells 7–11** — RAG: `USER_AGENT` for HTTP, `WebBaseLoader`, splitters, embeddings, `retriever`, `init_chat_model`, `@traceable` `rag_bot`.  
4. **Cell 12** — RAG examples dataset: **get-or-create** by `rag_dataset_name`, `create_examples`.  
5. **Cells 14–17** — RAG evaluators (structured outputs via `init_chat_model` + JSON schema).  
6. **Cell 18** — `rag_target(inputs) → rag_bot(inputs["question"])`, `evaluate` with `data=rag_dataset_name`.

**Important:** `groundedness` and `retrieval_relevance` read retrieved text from **`outputs["documents"]`** (what `rag_bot` returns), not from dataset inputs.

---

## Datasets and 409 conflicts

LangSmith dataset names are **unique per workspace**. The notebook uses:

- `try: create_dataset(...) except LangSmithConflictError: read_dataset(dataset_name=...)`

so re-running a dataset cell does not crash with **409 Conflict**.

**Caveat:** `create_examples` **appends** rows. Re-running that part repeatedly duplicates examples unless you skip it or clean the dataset in the UI.

---

## Evaluators (quick reference)

**Chatbot track**

- **Correctness** — LLM compares model answer to reference; boolean → score 0/1 in LangSmith.  
- **Concision** — Float 0–1 from reference vs prediction word counts (`min(1, ref_words / pred_words)`).

**RAG track**

- **Correctness** — Answer vs gold reference (structured grader).  
- **Relevance** — Answer vs user question.  
- **Groundedness** — Answer supported by **retrieved** `Document.page_content`.  
- **Retrieval relevance** — Retrieved chunks relevant to the question.

Evaluators use the LangSmith signature **`(inputs, outputs, reference_outputs)`** (some allow optional `reference_outputs`).

---

## Reading results in LangSmith

Open the **experiment** link printed after `evaluate`. You’ll see:

- Inputs, reference outputs, model outputs  
- Per-evaluator scores and aggregates  
- Latency, tokens, and cost when calls are traced  

If scores look “too low,” check whether **references** match how you want the model to behave (e.g. short marketing copy vs long factual answers) and whether judges’ prompts are too strict.

---

## Troubleshooting

| Issue | What to check |
|-------|----------------|
| `OPENAI_API_KEY` / 401 | Key in `.env`; run the env cell before OpenAI calls. |
| `gpt-40-mini` or model 404 | Model id is **`gpt-4o-mini`** (letter **o**, not zero **0**). |
| `ModuleNotFoundError: bs4` | `pip install beautifulsoup4`. |
| `USER_AGENT` warning | RAG cell sets a default `USER_AGENT` for `WebBaseLoader`. |
| `NameError` on evaluators | Run evaluator cells **before** the final `evaluate` cell. |
| `409` on dataset create | Should be handled by get-or-create; use a new name if you want a fresh dataset. |
| `inputs` vs `input` | Target and evaluators must use the parameter name **`inputs`** and match keys (`question`, `answer`). |

---

## Files

| File | Role |
|------|------|
| `rag-evaluation.ipynb` | Full workflow: env, datasets, apps, RAG, evaluators, experiments. |
| `requirements.txt` | Pinned-style list of Python dependencies. |
| `.env` | Secrets (local only; keep out of git). |
| `README.md` | This guide. |

---

## Glossary

- **Dataset** — Named collection of examples (inputs + optional reference outputs) in LangSmith.  
- **Example** — One row in a dataset.  
- **Experiment** — One evaluation run over a dataset with a target function and evaluators.  
- **Evaluator** — Function that scores a single prediction (often returns `bool`, `float`, or a dict with `score`).  
- **LLM-as-judge** — Using a model with a rubric (or structured schema) to grade outputs.  
- **RAG** — Retrieve context from a vector store, then generate an answer conditioned on that context.

---

## Links

- [LangSmith docs](https://docs.smith.langchain.com/)  
- [LangSmith evaluation guide](https://docs.smith.langchain.com/evaluation)  
- [OpenAI API](https://platform.openai.com/docs)
