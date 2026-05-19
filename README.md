# Ask Craig

A Corrective RAG (CRAG) chatbot that answers questions from local PDFs and falls back to live web search when local documents are insufficient. It matters because standard RAG systems generate confident answers even when their retrieved context is irrelevant — Ask Craig adds an AI judge that detects this and reroutes to the web, so answers are always grounded in something real.

## System diagram
![System diagram](assets/diagrams/system_diagram.png)

The diagram has four main components. The **Retriever** splits PDFs into overlapping text chunks, embeds them with Gemini, and stores them in a FAISS vector index for fast similarity search. The **Evaluator** asks Gemini to read the retrieved chunks and return a single verdict — `relevant`, `irrelevant`, or `ambiguous` — which controls where the answer comes from. The **Web Search** module calls Tavily when local documents aren't sufficient. The **Generator** receives whichever context won the routing decision and produces the final answer. All of this is orchestrated by `main.py`, which also ensures the PDF index is built exactly once per session.

## Screenshot
![Project Showcase](assets/project_screenshot.png)

## How it works

1. User asks a question via the Gradio chat UI.
2. The system retrieves the most relevant chunks from indexed local PDFs.
3. A Gemini model judges whether the retrieved context is `relevant`, `irrelevant`, or `ambiguous`.
4. Depending on the verdict:
   - **Relevant** → answer from local documents only.
   - **Irrelevant** → answer from Tavily web search.
   - **Ambiguous** → answer from both sources combined.

## Setup

### 1. Clone the repo and enter the directory

```bash
git clone <repo-url>
cd Applied-AI-System
```

### 2. Create a virtual environment and install dependencies

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 3. Configure API keys

Copy `.env.example` to `.env` and fill in your keys:

```bash
cp .env.example .env
```

| Key | Where to get it |
|-----|-----------------|
| `GEMINI_API_KEY` | [Google AI Studio](https://aistudio.google.com/app/apikey) |
| `TAVILY_API_KEY` | [Tavily](https://app.tavily.com/) |

### 4. Add your PDFs

Place any PDF files you want to query in the `data/` folder:

```
data/
  your-document.pdf
  another-file.pdf
```

If the folder is empty the system will skip local retrieval and fall back to web search for every query.

### 5. Run the app

```bash
python app.py
```

Open the URL printed in the terminal (default: `http://127.0.0.1:7860`).

To run without the UI (terminal only):

```bash
python -m src.main
```

## Project structure

```
app.py              # Gradio web interface entry point
src/
  clients.py        # API client setup (Gemini, Tavily)
  retriever.py      # PDF loading, chunking, embedding, FAISS index
  evaluator.py      # Relevance scoring via Gemini
  generator.py      # Final answer generation via Gemini
  web_tools.py      # Tavily web search fallback
  main.py           # CRAG pipeline orchestration
data/               # Place your PDF files here
requirements.txt
.env.example        # Copy to .env and add your keys
```

## Design decisions

**Why add an evaluator instead of always using web search?**
Always trusting local documents causes hallucinations when a question falls outside them; always using the web ignores files the user explicitly uploaded. The evaluator routes intelligently at the cost of one extra API call per query.

**Why FAISS instead of a managed vector database?**
FAISS runs in-process with no setup. The trade-off is that the index is rebuilt from scratch each startup; a persistent store like Chroma would be worth it at a larger scale.

**Why Gemini for both embedding and generation?**
One provider means one API key and a smaller dependency surface. The trade-off is vendor lock-in — swapping models requires edits in multiple files.

**Why Gradio for the UI?**
Gradio's `ChatInterface` delivers a working chat UI in ~10 lines with no frontend code. The trade-off is limited styling and layout control.

**Why split into five files?**
Each module has one job and can be tested in isolation. A failing test points immediately to the broken pipeline stage.

## Running the tests

```bash
# from the project root, with your venv active
pytest tests/ -v
```

No API keys or PDF files are needed — every external call is mocked. The suite covers:

| File | What is tested |
|------|---------------|
| `tests/test_retriever.py` | Text chunking, empty-index guard, PDF indexing, corrupt-PDF skip |
| `tests/test_evaluator.py` | All three verdicts, unexpected model output, API failure fallback |
| `tests/test_generator.py` | Normal answer, source label in prompt, API failure message |
| `tests/test_web_tools.py` | Result formatting, empty results, API failure |
| `tests/test_main.py` | Empty query, relevant/irrelevant/ambiguous routing, empty-web fallback, unhandled exception |


## Logging

The app logs to stdout at `INFO` level. Each pipeline stage (indexing, retrieval, relevance verdict, generation) emits a log line so you can trace what happened for any given query.
