# Chatbot evaluation with LangSmith

This project is a **small lab notebook** that checks how well a simple OpenAI chatbot answers fixed questions. It uses **LangSmith** to store test questions, run the bot on each one, and score the answers with automatic metrics.

You do **not** need a separate “RAG” index here—the focus is on **evaluation**: dataset → model → scores in the LangSmith UI.

---

## What you need

- Python 3 with Jupyter (or VS Code / Cursor with notebook support)
- A **LangSmith** account and API key ([LangSmith](https://smith.langchain.com))
- An **OpenAI** API key

Install packages (example):

```bash
pip install python-dotenv langsmith openai jupyter
```

---

## Configuration (`.env`)

Create a `.env` file in this folder (do **not** commit real keys to git):

| Variable | Purpose |
|----------|---------|
| `OPENAI_API_KEY` | Used by the OpenAI client (preferred name) |
| `OPEN_API_KEY` | Optional fallback—the notebook copies this to `OPENAI_API_KEY` if the standard name is missing |
| `LANGSMITH_API_KEY` | Lets the notebook talk to LangSmith |

The first notebook cell also turns on tracing with `LANGSMITH_TRACING=true` so runs can show up in LangSmith.

---

## How the notebook is organized

Open **`rag-evaluation.ipynb`** and run cells **from top to bottom** after setting `.env`.

### 1. Load keys and environment

- Loads `.env` with `dotenv`.
- Puts API keys into `os.environ` so `openai.OpenAI()` and `langsmith.Client()` work automatically.

### 2. Create a dataset in LangSmith

- Builds a **LangSmith Client** and a dataset named `MychatBot` (change the name if you want a new dataset).
- Adds **examples**: each row has:
  - **inputs**: e.g. `{"question": "..."}`
  - **outputs**: a **reference answer** (“gold” text you consider a good answer for grading).

**Note:** Running this cell again may try to create the same dataset again. If the dataset already exists, you might need to skip this cell or use a new dataset name—otherwise you can get errors from LangSmith.

### 3. Custom metrics (evaluators)

Evaluators are plain Python functions LangSmith calls with three dictionaries:

- `inputs` — the question (and anything else you put in the example’s inputs)
- `outputs` — what **your app** returned for that row
- `reference_outputs` — the gold answer from the dataset

**Correctness (LLM as judge)**  
Another model (`gpt-4o-mini`) reads the question, your answer, and the reference. It replies YES or NO following the system instructions (correct/helpful vs wrong/off-topic). The code turns that into `True` / `False`, which LangSmith shows as **1** or **0**.

**Concision**  
Measures how **short** the answer is compared to the reference, as a score from **0 to 1**: if the model answer is much longer than the reference, the score goes down; if it’s similar length or shorter, the score stays high.

The OpenAI client is wrapped with `langsmith.wrappers.wrap_openai` so those API calls can be traced in LangSmith.

### 4. The app under test: `my_app`

- Takes a **question string**, calls the chat API with a system prompt, and returns the assistant’s **text** only.
- This is the “product” you are evaluating.

### 5. Adapter: `ls_target`

LangSmith expects a function that receives **`inputs`** (a dict, e.g. `{"question": "..."}`) and returns **`outputs`** as a dict.

Here it returns `{"answer": ...}` so the keys match the reference format (`answer`) and the evaluators can read `outputs["answer"]`.

### 6. Run the evaluation

`client.evaluate(...)`:

- Runs `ls_target` on **every row** of the dataset `MychatBot`.
- Runs **correctness** and **concision** on each run.
- Prints a link to an **experiment** in the LangSmith UI (latency, tokens, scores, side‑by‑side comparison).

---

## Reading results in LangSmith

In the experiment view you typically see:

- **Inputs** — the question  
- **Reference outputs** — your gold answer  
- **Outputs** — what `my_app` actually said  
- **correctness** — average reflects how often the judge said YES  
- **concision** — average reflects typical brevity vs reference  
- **Latency / tokens / cost** — from traced OpenAI calls  

Low correctness often means the judge is strict or the reference doesn’t match how you want the real model to answer (e.g. short marketing lines vs long factual replies). You can change the judge prompt or the reference answers to match your goals.

---

## Files in this repo

| File | Role |
|------|------|
| `rag-evaluation.ipynb` | All code: setup, dataset, evaluators, app, `evaluate` |
| `.env` | Your secrets (local only) |
| `README.md` | This explanation |

---

## Quick glossary

- **Dataset** — A table of test cases in LangSmith (inputs + optional reference outputs).  
- **Example** — One row in that table.  
- **Experiment** — One full pass: run your function on every example and attach evaluator scores.  
- **Evaluator** — A function that scores one run (correctness, concision, etc.).  
- **LLM-as-judge** — Using a second model to label quality instead of hand-written code only.

If you want to extend this later, common next steps are: more examples, multiple models, or swapping `my_app` for a real RAG pipeline while keeping the same `ls_target` shape and evaluators.
