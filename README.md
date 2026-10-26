# AI Customer Support System

A real-time AI support assistant that answers customer questions from a company's own documents (FAQs, policies, manuals) using **Retrieval-Augmented Generation (RAG)**. Answers stream word-by-word over **WebSockets**, cite their sources, and repeated questions are served instantly from a **Redis** cache.

![Tech](https://img.shields.io/badge/stack-React%20%7C%20FastAPI%20%7C%20LangChain%20%7C%20Pinecone%20%7C%20Redis-blue)

## Features
- Upload knowledge – PDF, TXT or Markdown files are split into chunks, embedded with OpenAI, and stored in Pinecone.
- Grounded answer – the LLM only answers from retrieved context and says "I don't know" instead of making things up.
- Real-time streaming – tokens stream to the browser over a WebSocket, with auto-reconnect.
- Source citations + latency** shown under every answer.
- Redis caching** – normalized questions are cached; the cache clears when new documents are uploaded.
- Metrics endpoint – questions served, cache hit rate, average latency, active connections.
- Production setup – Docker Compose, Kubernetes manifests with autoscaling (HPA), health probes, GitHub Actions CI.
- Load testing script for concurrent WebSocket users.

## Architecture
```
 React (Vite, TS) ──WebSocket──▶ FastAPI ──▶ Redis (answer cache)
        │                          │
        └──── POST /documents ─────┤
                                   ├──▶ OpenAI embeddings ──▶ Pinecone (vector search)
                                   └──▶ GPT-4o-mini (streamed answer, grounded in retrieved chunks)
```

Request flow:** question → check Redis → (miss) embed question → top-k similar chunks from Pinecone → prompt LLM with chunks → stream tokens to client → cache answer.

## Project structure
```
backend/
  app/config.py     settings from environment variables
  app/rag.py        document loading, chunking, Pinecone storage, retrieval, streaming answers
  app/cache.py      Redis answer cache (works without Redis too)
  app/main.py       FastAPI routes: /documents, /ws/chat, /health, /metrics
  tests/            pytest tests using fakes (no API keys needed)
  sample_docs/      example FAQ to try it out
frontend/src/
  useChat.ts        WebSocket hook: streaming, reconnect, message state
  App.tsx           chat UI + document upload
k8s/deployment.yaml backend Deployment, Service, HPA (2–10 pods) and Redis
loadtest/           concurrent WebSocket load test
```

## Run locally

1. Keys:** copy `.env.example` to `.env` and add your OpenAI and Pinecone API keys (Pinecone free tier works; the index is created automatically).

2a. With Docker (easiest):
```bash
docker compose up --build
```
Frontend: http://localhost:3000 · API docs: http://localhost:8000/docs

2b. Without Docker:
```bash
# Redis (optional – app works without it, just no caching)
docker run -p 6379:6379 redis:7-alpine

# Backend
cd backend
python -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload

# Frontend (new terminal)
cd frontend
npm install
npm run dev        # http://localhost:5173
```

3. Try it: upload `backend/sample_docs/company_faq.md`, then ask "How long does shipping take?" or "Can I return a sale item?"

## Tests
```bash
cd backend && pytest -q
```

## Load test
```bash
pip install websockets
python loadtest/load_test.py --users 100 --questions 3
```
Prints average and p95 latency and throughput. Ask the same questions twice to see the Redis cache effect.

## Deploy on Kubernetes
```bash
docker build -t ai-customer-support-backend:latest ./backend
kubectl create secret generic support-secrets \
  --from-literal=OPENAI_API_KEY=sk-... --from-literal=PINECONE_API_KEY=pcsk_...
kubectl apply -f k8s/deployment.yaml
```
The HorizontalPodAutoscaler scales the backend from 2 to 10 pods at 70% CPU.

## API
| Method | Path | Description |
|---|---|---|
| POST | `/documents` | Upload PDF/TXT/MD (max 10 MB) |
| WS | `/ws/chat` | Send `{"question": "..."}`; receive `token` messages then `done` with sources and latency |
| GET | `/metrics` | Usage, cache hit rate, average latency |
| GET | `/health` | Health check |

## Design decisions
- RAG instead of fine-tuning: company docs change often; RAG updates instantly by re-uploading files.
- Chunk size 800, overlap 120: keeps chunks focused while preserving context across boundaries.
- WebSockets instead of HTTP polling:** lower latency and true token streaming.
- Cache key = normalized question hash:** "Can I return?" and "can i   RETURN?" hit the same entry.
- Graceful degradation:** if Redis is down, the app still answers.

## Future improvements
- Hand-off to a human agent when the AI is unsure
- Conversation memory for follow-up questions
- Authentication and multi-tenant indexes per company
- Semantic (embedding-based) cache
