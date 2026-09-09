# TutorAgentPoC

A proof-of-concept grammar tutor that runs **entirely on your own machine**. A local LLM
served by [Ollama](https://ollama.com) reads a learner's text, decides which grammar topics
it violates, looks those topics up in a real grammar book, and returns a structured summary
of the broken rules — without ever pointing at *where* in the text the errors are, so the
learner has to find them.

| Notebook | What it is |
| --- | --- |
| `EnglishTutorPoC.ipynb` | The main PoC. Grounded in *New Round-Up 5 – English Grammar Practice* (Evans & Dooley, Pearson). |
| `GermanTutorPoC.ipynb` | The earlier, simpler German sketch the English one was modeled on. |
| `one pager/` | Project one-pager slides (Marp), unrelated to running the tutor. |

Nothing leaves your laptop: no API keys, no cloud calls.

---

## 1. Prerequisites

- **macOS, Linux or Windows** with ~20 GB free disk space
- **~24 GB RAM** to run the default `gemma4:26b` model comfortably (see
  [Using a smaller model](#using-a-smaller-model) if you have less)
- **Python 3.9+**
- A copy of the **New Round-Up 5 PDF** (see [step 4](#4-supply-the-grammar-reference-pdf) — it
  is deliberately not in the repo)

---

## 2. Install and start Ollama

Ollama runs the language model locally and exposes an OpenAI-compatible HTTP API on
`http://localhost:11434`, which is exactly what the notebook talks to.

### Install

**macOS (Homebrew)**
```bash
brew install ollama
```

**macOS / Windows (installer)**
Download the app from <https://ollama.com/download> and run it. On Windows the installer
starts the background service for you.

**Linux**
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### Start the server

```bash
ollama serve
```

Leave this running in its own terminal window. If you installed the macOS/Windows desktop
app instead, launching the app already starts the server and `ollama serve` will just tell
you the address is in use — that's fine.

On Linux, the install script registers a systemd service, so you can use:
```bash
sudo systemctl start ollama
```

### Pull the model

In a second terminal:

```bash
ollama pull gemma4:26b
```

This downloads ~17 GB once. Check what you have:

```bash
ollama list
```

### Verify it works

```bash
curl http://localhost:11434/v1/models
```

You should get a JSON list containing `gemma4:26b`. If this command fails, the notebook
will fail too — fix it here first.

### Using a smaller model

`gemma4:26b` is the default because larger models are markedly better at reliably producing
the **structured JSON output** the notebook demands. If you don't have the RAM:

```bash
ollama pull llama3.1:8b
```

and change `model_name` in the model cell of the notebook. Expect more retries and
occasional validation failures — the agents are configured with `retries=3` for this reason.

---

## 3. Set up the Python environment

From the repository root:

```bash
python3 -m venv .venv
source .venv/bin/activate           # Windows: .venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt
```

`requirements.txt` pins the exact versions this PoC was verified against (Python 3.9.6):

- **pydantic-ai / pydantic** — the agent framework; enforces that the model's answer matches a Pydantic schema
- **openai** — the HTTP client pydantic-ai uses to reach Ollama's OpenAI-compatible endpoint
- **pypdf** — extracts the grammar book's text
- **langdetect** — rejects non-English submissions before they reach the model
- **ipykernel** — the Jupyter kernel, needed to run the notebook from VS Code

Versions are pinned exactly because `pydantic-ai` is still pre-1.0 and its agent/output API
changes between minor releases — and that API is exactly what the notebook's structured
output relies on. To run the notebook in a browser rather than in VS Code, also
`pip install jupyterlab` (left out of the pins on purpose; see the comments in the file for
a lighter-weight `pydantic-ai-slim[openai]` alternative too).

---

## 4. Supply the grammar reference PDF

The book PDF and its extracted cache are listed in `.gitignore`, so a fresh clone does not
contain them. Put your copy here:

```
data/New-Round-Up-5.pdf
```

On the first run the notebook parses the PDF into text chunks and caches them as JSON, so
subsequent runs start instantly.

> **Note on the cache path:** the notebook writes and reads
> `new_round_up_5_reference.json` in the **repository root**, while an already-extracted
> copy ships at `data/new_round_up_5_reference.json`. If you have that file and want to skip
> PDF parsing entirely, either copy it to the root or change `REFERENCE_PATH` in the
> extraction cell to `Path("data/new_round_up_5_reference.json")`.

---

## 5. Start the notebook

Pick whichever you prefer — all three run the same code.

**JupyterLab (browser)**
```bash
source .venv/bin/activate
jupyter lab
```
Your browser opens at `http://localhost:8888`; click `EnglishTutorPoC.ipynb`.

**Classic Notebook**
```bash
jupyter notebook EnglishTutorPoC.ipynb
```

**VS Code**
Open the folder, open `EnglishTutorPoC.ipynb`, and in the top-right kernel picker choose the
`.venv` interpreter. VS Code will prompt to install `ipykernel` if it's missing.

Then run the cells **top to bottom** (`Shift+Enter`, or *Run All*). Order matters — later
cells depend on names defined earlier.

**Checklist before running:** Ollama serving ✅ · model pulled ✅ · PDF in `data/` ✅ · `.venv` kernel selected ✅

---

## 6. What each cell does

### Setup

| Cell | Purpose |
| --- | --- |
| **Imports** | Pulls in `Agent` plus the OpenAI model/provider classes that will be pointed at Ollama. |

### 1. Build a searchable reference from the PDF

Reads `data/New-Round-Up-5.pdf` page by page, drops watermark lines and near-empty pages,
and slices the text into **1200-character chunks with a 200-character overlap** (the overlap
keeps a rule and its examples from being split across a boundary). The result is cached to
JSON so the 200+ page PDF is parsed only once.

### 2. Lightweight keyword search

Builds a token `Counter` for every chunk and defines `search_reference(query, top_k)`, which
scores chunks by keyword overlap and returns the best ones with page numbers. Deliberately
**no embedding model** — for a PoC, keyword overlap is enough to ground explanations in the
book's own wording, and it keeps the dependency list tiny.

### Model configuration

Points `OpenAIModel` at `http://localhost:11434/v1`. This is the one cell you edit to swap
models. **If Ollama isn't running, this cell still succeeds** — the failure surfaces later,
on the first actual agent call.

### Output schemas

Three Pydantic models — `TopicDetection`, `Violation`, `GrammarSummary` — define the JSON
contract. pydantic-ai turns these into a tool schema the model must call, so the notebook
gets typed objects back instead of prose it would have to parse.

### The two agents

Splitting the job in two keeps each prompt narrow:

- **`topic_agent`** — reads the text and names only the violated grammar topics, from the
  fixed New Round-Up 5 chapter list. No explanations.
- **`explain_agent`** — receives the text *plus* the reference material and writes the
  explanations. Its prompt forbids revealing error locations and forbids reusing the
  learner's own sentences as examples, so the learner still has to hunt.

### Agent memory tools

`memory_search` and `store_errors_in_memory`, registered on `explain_agent` as callable
tools backed by an in-process list. A placeholder for real per-learner persistence.

### 2b. Control flow lives in Python, not in prose

The key design cell. Eight behaviours that used to be *requests* in the system prompt —
iteration cap, stop words, language check, long-text splitting, clustering repeats, always
consulting the reference — are now Python. A 26B local model asked politely to count its
iterations is not a control; a counter is.

| Function | Enforces |
| --- | --- |
| `fetch_reference_for_topics` | The lookup **always** runs; the model can't skip it. |
| `check_chunk` | Fixes the order: detect topics → fetch reference → explain. |
| `is_stop_keyword`, `is_english` | `stop`/`exit` and non-English input are rejected before any model call. |
| `split_into_chunks` | Texts over 250 words are processed in pieces. |
| `cluster_violations` | Repeats of the same rule collapse into one entry with a summed count. |
| `TutorSession` | Counts submissions of the same text in Python and caps them at 5. |

`check_text` is the single entry point that runs those guards, loops over chunks, clusters
the result, and averages the confidence scores.

### 3. First iteration — run on flawed English

Creates a `TutorSession` and submits a sample text seeded with errors across several
chapters (Present Perfect vs Past Simple, Past Continuous, modal `must`, articles), then
prints the structured summary. **This is the first cell that actually contacts Ollama, so
it's the slow one and the one that fails if the server is down.** Expect tens of seconds on
a 26B model.

### 4. Iterative correction loop

The intended classroom workflow: a `render` helper formats the summary for a human, then the
same session takes turn 1 (flawed text) and turn 2 (the learner's corrected attempt). The
list of violations should visibly shrink between turns — that shrinking is the PoC's actual
success criterion.

---

## 7. Trying your own text

After running all cells, add a new cell at the bottom:

```python
session = TutorSession()
result = await check_text(session, "Your English text goes here.")
print(render(result))
```

`await` works directly because Jupyter already runs an event loop.

---

## 8. Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `APIConnectionError` / `Connection refused` | Ollama isn't running. Start `ollama serve` and re-run `curl http://localhost:11434/v1/models`. |
| `model 'gemma4:26b' not found` | Run `ollama pull gemma4:26b`, or point `model_name` at a model from `ollama list`. |
| `FileNotFoundError: data/New-Round-Up-5.pdf` | The PDF is gitignored — see [step 4](#4-supply-the-grammar-reference-pdf). |
| Very slow, machine swapping | The 26B model doesn't fit in RAM. Switch to `llama3.1:8b`. |
| Validation / retry errors from pydantic-ai | The model returned prose instead of calling the output function. Common on smaller models; re-run, or use a larger one. |
| `Input is not English` on valid text | `langdetect` is unreliable on very short strings. Submit a longer passage. |
| `NameError` on a helper function | Cells were run out of order. Use *Run All* from the top. |

---

## License

See [LICENSE](LICENSE).
