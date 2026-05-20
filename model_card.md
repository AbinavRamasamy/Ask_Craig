# Applied AI System Model Card

## Models used

| Role | Model | Provider | Notes |
|------|-------|----------|-------|
| Text embedding | `text-embedding-004` | Google Gemini | Converts PDF chunks and queries into 768-dimensional vectors for similarity search |
| Relevance evaluation | `gemma-4-31b-it` | Google Gemini | Reads retrieved context and returns `relevant`, `irrelevant`, or `ambiguous` |
| Answer generation | `gemma-4-31b-it` | Google Gemini | Produces the final grounded answer from whichever context won the routing decision |
| Web search | Tavily Search API | Tavily | Returns top-3 result snippets with URLs when local documents are insufficient |

---

## Intended use

**Primary use case:** Answering questions about documents a user has uploaded locally (PDFs), with automatic fallback to live web search when those documents don't contain the answer.

**Intended users:** Students, researchers, or professionals who want to query their own documents in natural language without needing to read every page.

**Intended environment:** Local or personal deployment. The app runs on a single machine with two free-tier API keys.

---

## Out-of-scope uses

- **Not suitable for high-stakes decisions** — medical, legal, or financial advice should not rely on this system without expert review.
- **Not designed for real-time or safety-critical systems** — API latency (typically 2–5 seconds per query) makes it unsuitable for time-sensitive applications.
- **Not a substitute for primary source verification** — the system reports what its sources say; it does not independently fact-check those sources.
- **Not multilingual** — performance degrades for non-English documents and queries because the underlying models are optimized for English.

---

## System pipeline

```
User query
    │
    ▼
Retriever — embed query → FAISS nearest-neighbor search → top-3 chunks from PDFs
    │
    ▼
Evaluator — Gemini judges: relevant / irrelevant / ambiguous
    │
    ├── relevant   → Generator (local context only)
    ├── irrelevant → Web Search → Generator (web context only)
    └── ambiguous  → Web Search → Generator (local + web combined)
    │
    ▼
Final answer returned to user via Gradio chat UI
```

---

## Training data

This system does not train any models. It uses pre-trained models from Google Gemini via API. The document index is built at runtime from PDFs placed in the `data/` folder by the user — no data leaves the user's machine except as API requests to Google and Tavily.

---

## Performance and evaluation

**No formal benchmark evaluation was conducted.** System behavior was validated through:

- Manual testing across the three routing paths (relevant, irrelevant, ambiguous)
- A unit test suite (23 tests, 100% pass rate) covering all pipeline modules with mocked API calls
- Observed routing behavior: the `ambiguous` verdict fires more frequently than expected when relevant information is distributed across multiple document chunks rather than concentrated in one

**Known performance factors:**

| Factor | Effect |
|--------|--------|
| Chunk size (400 words) | Larger chunks capture more context per retrieval but dilute relevance scoring |
| Top-k retrieval (k=3) | Retrieving more chunks improves recall but increases evaluator confusion |
| Evaluator temperature (0) | Deterministic verdict reduces randomness but may be overly cautious |
| Generator temperature (0.3) | Slight creativity in phrasing while staying grounded in context |

---

## Limitations

**Document quality dependence:** The system can only be as accurate as the PDFs it indexes. Scanned PDFs with poor OCR, outdated documents, or documents written from a particular bias will produce answers that reflect those flaws.

**No contradiction resolution:** When routing is `ambiguous`, local and web context are concatenated without any mechanism to detect or resolve conflicts between the two sources.

**No source ranking:** Web results are returned in Tavily's default order. The system does not evaluate the credibility or recency of web sources before including them in the answer.

**Memory:** The FAISS index is rebuilt from scratch on every startup. No state is persisted between sessions.

**Chunk boundary artifacts:** Splitting documents into fixed-size word windows can cut sentences mid-thought. A retrieved chunk may start or end in the middle of an argument, reducing coherence.

---

## Ethical considerations and potential misuse

**Misinformation amplification:** If a user loads documents containing false or misleading claims, the system will answer based on that content and present it with the same confidence as accurate information. There is no built-in fact-checking layer.

**Sensitive content:** The system has no content moderation on inputs or outputs. Users can query sensitive topics, and the generator will respond based on whatever context it receives.

**Mitigation recommendations for deployment:**
- Add an output content filter before displaying answers to users
- Display source citations prominently so users can verify claims
- Add a disclaimer that answers should not be used as a substitute for professional advice
- Implement per-user rate limiting to prevent API abuse via the web search fallback

---

## Caveats and recommendations

- Always review answers against the cited source before acting on them
- For best results, use clean, text-based PDFs rather than scanned images
- If the system consistently routes to web search for questions you expect the document to answer, the document may have poor text extraction — try a different PDF export tool
- The system is designed for personal/educational use; a production deployment would require additional safety layers, persistent storage, and API cost management

---

## Testing summary

**What worked well:** Mocking at the module level (`@patch("src.evaluator.google_client")`) made tests fast and fully deterministic — no real API calls, no flakiness, no keys required. Testing pure functions like `_chunk_text` required no mocking at all. The `conftest.py` approach of patching SDK constructors before any `src.*` import cleanly solved the problem of `clients.py` validating keys at import time.

**What was tricky:** Getting the `conftest.py` import order right took iteration — `os.environ` must be set and the SDK constructors must be patched *before* any test file imports a `src.*` module, otherwise the real clients try to initialize. Also, `sys.path` had to be explicitly set because pytest doesn't automatically treat the project root as importable.

**What isn't tested:** The Gradio UI layer is not covered — verifying the chat interface renders correctly requires a browser. End-to-end integration tests with live API calls are also absent; the mock-based unit tests verify logic but not that the APIs return usable responses in practice.

---

## Reflection

**Limitations and biases**
The system is only as accurate as the PDFs it indexes — outdated or biased documents produce equally skewed answers with no warning to the user.
Tavily's ranking silently favors certain sources, and the `ambiguous` path blends local and web context with no way to detect contradictions between them.
Both models also perform best in English, so multilingual queries may return noticeably weaker results.

**Potential for misuse**
Loading documents with false claims lets the system present misinformation as a confident, sourced answer.
There is also no content moderation on inputs or outputs, so users can query sensitive topics without restriction.
A deployed version should add output filtering and a disclaimer that answers need independent verification.

**What surprised me during reliability testing**
The `ambiguous` verdict activated far more often than expected — relevant information spread across multiple chunks was frequently misclassified, triggering unnecessary web searches.
This added latency and occasionally pulled in web results that contradicted the uploaded document.
It made clear that chunk size and top-k retrieval are more critical tuning parameters than they initially seemed.

**Collaboration with AI during this project**
AI helped throughout with code structure, error handling, and building the test cases.
One helpful suggestion: patch mocks at the module where a client is *used* (`@patch("src.evaluator.google_client")`), not where it's defined — otherwise the mock doesn't intercept the actual call.
One flawed suggestion: the initial python test files were missing required modules, causing every test to fail with `ModuleNotFoundError`. This reminded me to always run generated code before trusting it.
