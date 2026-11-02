# Qubit-Chat

Chat with your PDFs, with a retrieval step that re-ranks candidate passages using **Grover's search** on Qiskit's Aer simulator.

Upload a document, ask a question and Qubit-Chat finds the passages that matter, amplifies the strongest matches with a Grover circuit built for that query and hands them to Gemini to write a grounded answer. A general chat mode works without any documents.

## How retrieval works

```
PDF ─► pypdf text ─► 1,000-char chunks (200 overlap) ─► MiniLM embeddings ─► ChromaDB
                                                                              │
question ─► MiniLM embedding ─► cosine similarity against the user's chunks ◄─┘
                                        │
        chunks above the similarity threshold become the "marked" items
                                        │
              Grover circuit: ⌈log₂ N⌉ qubits, oracle + diffuser, 1,024 shots
                                        │
           score = similarity × (1 + 2.0 × measured probability) ─► top-k ─► Gemini
```

- **The threshold depends on the caller:** 0.7 for the search API and a looser 0.3 for chat context.
- **Oracle and diffuser are built per query.** The oracle phase-flips the indices of the marked chunks with a multi-controlled Z gate; the diffuser amplifies them. The iteration count is the textbook ⌊π/4 · √(N/M)⌋, capped at 10.
- **Quantum boosts the score; it doesn't replace it.** Measured probabilities are folded into the classical cosine score, so a strong semantic match always stays near the top.
- **It falls back instead of failing.** If no chunk clears the threshold, the collection needs more than 10 qubits (over 1,024 chunks) or the circuit returns nothing, ranking drops to plain cosine similarity. Every response reports which path was used (`quantum_enhanced` or `classical_fallback`) and why.
- **Structural noise is filtered.** Page separators and near-empty chunks are pushed below informative passages before the top-k is taken.

This runs on a simulator, so it's an exploration of quantum-enhanced ranking rather than a speed-up claim: the classical similarity pass already touches every chunk.

## Features

- PDF upload with per-user, per-session document scoping in ChromaDB
- Document-grounded chat and a general chat mode, both answered by Gemini
- Sign-up and login with salted password hashing; users and chat sessions stored in Supabase
- Search stats and "similar documents" endpoints for inspecting retrieval
- 20 backend tests covering Grover retrieval, every fallback path, Chroma filter building and auth configuration

## Stack

| Layer | Tech |
|---|---|
| Quantum | Qiskit 0.46, Qiskit Aer |
| Retrieval | sentence-transformers (`all-MiniLM-L6-v2`), ChromaDB, LangChain text splitting |
| LLM | Google Gemini |
| Backend | FastAPI, Pydantic, Supabase |
| Frontend | React, TypeScript, Vite, Tailwind CSS, shadcn/ui |
| Deployment | Dockerfiles for both services, nginx for the frontend |

## Run it locally

**Backend** (Python 3.10+)

```bash
cd backend
python -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp ../.env.example .env                           # add GEMINI_API_KEY, SUPABASE_URL, SUPABASE_ANON_KEY
uvicorn main:app --reload --port 8000
```

API docs: http://localhost:8000/docs

**Frontend** (Node 18+)

```bash
cd frontend
npm install
echo "VITE_API_BASE_URL=http://localhost:8000" > .env.local
npm run dev
```

Open http://localhost:8080

**Tests**

```bash
cd backend
python -m pytest tests -q
```

## API

| Method | Path | What it does |
|---|---|---|
| `POST` | `/api/v1/upload_pdf` | Upload, chunk and embed a document |
| `POST` | `/api/v1/query` | Quantum-enhanced search over the user's chunks |
| `GET` | `/api/v1/search_stats` | Retrieval statistics |
| `GET` | `/api/v1/similar_documents/{id}` | Nearest neighbours of a chunk |
| `GET` | `/api/v1/pdf_stats` | Indexed document stats |
| `DELETE` | `/api/v1/clear_pdfs` | Clear indexed documents |
| `POST` | `/api/chat` | Grounded or general chat |
| `POST` | `/register`, `/login` | Accounts |

## Configuration

Quantum settings live in `backend/config.py` and can be overridden with environment variables:

| Variable | Default | Meaning |
|---|---|---|
| `QUANTUM_SHOTS` | `1024` | Measurements per circuit |
| `QUANTUM_MAX_QUBITS` | `10` | Above this, ranking falls back to classical |
| `QUANTUM_BOOST_FACTOR` | `2.0` | Weight of the measured probability in the final score |
| `CHUNK_SIZE` / `CHUNK_OVERLAP` | `1000` / `200` | Characters per chunk and overlap |

More backend detail: [backend/README.md](backend/README.md).
