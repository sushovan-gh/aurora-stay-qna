# AuroraStay QnA

A retrieval-augmented question-answering assistant for **AuroraStay Hotels** staff. Front-desk and reservations staff type a guest question in plain English (cancellations, refunds, pets, parking, breakfast, accessibility, conduct) and get an answer grounded **only** in the hotel group's policy documents, together with the policy passages it was based on, so every answer can be checked before it is relayed to a guest.

This is an internal proof of concept: it is meant to measure how many staff queries can be answered reliably without manual lookup or escalation.

## Architecture

```
Policy PDFs (4)
   │  Docling: PDF -> structured markdown (tables preserved)
   ▼
Token-aware chunking (RecursiveCharacterTextSplitter)
   │  128-token chunks, 24-token overlap = the embedding model's full input window
   ▼
HarishMaths/Hotel-Policy-Embedding (hotel-policy sentence embeddings, normalized)
   ▼
FAISS vector index (162 chunks, serialized in backend/faiss_index)
   │  top-k similarity search (default k = 6)
   ▼
gpt-4o-mini (temperature 0) with grounding rules
   │  answer only from retrieved text; otherwise "not enough information"
   ▼
Flask REST API (gunicorn, port 7860)  <──HTTP──  Streamlit UI (port 8501)
```

## Repository layout

```
backend/
  app.py            Flask API: loads the embedding model and FAISS index once, exposes the endpoints below
  rag_utils.py      retrieve -> generate -> rag pipeline (same prompt rules as the evaluated notebook)
  faiss_index/      serialized FAISS index (index.faiss, index.pkl)
  requirements.txt
  Dockerfile        CPU-only PyTorch, embedding model downloaded at build time, gunicorn on 0.0.0.0:7860
frontend/
  app.py            Streamlit UI: question box, k / temperature / max-tokens sliders; shows the answer and the policy chunks behind it
  requirements.txt
  Dockerfile        Streamlit on 0.0.0.0:8501
```

## Run it in GitHub Codespaces

1. **Add Codespaces secrets** (repository **Settings → Secrets and variables → Codespaces → New repository secret**):
   - `OPENAI_API_KEY`: the API key
   - `OPENAI_API_BASE`: the OpenAI-compatible base URL
2. **Open a Codespace:** **Code → Codespaces → Create codespace on main**. Docker is available in the default Codespaces image.
3. **Backend** (in the Codespace terminal):
   ```bash
   cd backend
   docker build -t aurora-backend .
   docker run -d --name aurora-backend -p 7860:7860 \
     -e OPENAI_API_KEY -e OPENAI_API_BASE aurora-backend
   docker logs -f aurora-backend    # wait for "Vectors stored: 162", then Ctrl+C
   ```
4. **Frontend** (new terminal):
   ```bash
   cd frontend
   docker build -t aurora-frontend .
   docker run -d --name aurora-frontend --network host aurora-frontend
   ```
   `--network host` lets the Streamlit container reach the backend at `http://localhost:7860`.
5. **Ports:** in the **Ports** tab, right-click port **7860** → **Port Visibility → Public** (needed for API calls from outside the Codespace), and open port **8501** in the browser to use the UI.

## API

| Method | Endpoint | Body | Returns |
|---|---|---|---|
| GET | `/` | none | welcome message (health check) |
| POST | `/v1/relevant_chunks` | `{"query": "...", "k": 6}` | the top-k retrieved policy chunks |
| POST | `/v1/answer` | `{"query": "...", "k": 6, "model_name": "gpt-4o-mini", "temperature": 0, "top_p": 1, "max_tokens": 512}` | the grounded answer |
| POST | `/v1/answer_with_relevant_chunks` | same as `/v1/answer` | the answer plus the chunks it was based on |

Example:

```bash
curl -X POST "$BACKEND_URL/v1/answer_with_relevant_chunks" \
  -H "Content-Type: application/json" \
  -d '{"query": "Can a guest bring a dog that is not a service animal?", "k": 6}'
```

## Evaluation

Scored with DeepEval GEval metrics (gpt-4o as judge) on 10 evaluation and 10 held-out test questions.

| Metric | Evaluation set | Test set | Target |
|---|---|---|---|
| Faithfulness | 0.955 | 0.953 | ≥ 0.85 |
| Contextual Recall | 0.806 | 0.709 | ≥ 0.80 |
| Contextual Precision | 0.671 | 0.528 | ≥ 0.75 |
| Answer Relevancy | 0.793 | 0.719 | ≥ 0.75 |
| Composite (0.35 F + 0.25 CR + 0.20 CP + 0.20 AR) | 0.829 | 0.760 | > 0.75 for auto-answer |

- **15 of 20 questions (75%) qualify for automatic answers**, 10% for assisted review, 15% for manual lookup.
- Answers are highly faithful; the remaining misses are omissions (a single table row not retrieved), not invented policy.
- Chunk size and `k` were selected from a six-configuration sweep (`k = 3` composite 0.71 vs. `k = 6` composite 0.83).

## Notes

- Secrets are supplied only through environment variables / Codespaces secrets; `.env` and `config.json` are git-ignored.
- Delete the Codespace when finished to stop consuming resources.
