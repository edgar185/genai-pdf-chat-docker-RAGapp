# Chat With Your PDF — Containerized GenAI (RAG) on a Cloud VM

A containerized generative-AI application that lets you ask natural-language
questions about a PDF and get answers grounded in the document. Built as a set
of independent microservices orchestrated with Docker Compose on a Linux VM.

Ask *"What is biology?"* and get a coherent, document-grounded answer instead of
a keyword search — a Retrieval-Augmented Generation (RAG) pipeline end to end.

## Architecture

Every component runs in its own container. The model can be swapped without
touching the database or the front end — the microservices pattern used in real
enterprise AI deployments.

```
                    ┌─────────────────────────────────────────┐
   Browser  ─────▶  │  Streamlit UI  (port 8000)               │
                    │      │                                    │
                    │      ▼                                    │
                    │  LangChain orchestration                  │
                    │      │                  │                 │
                    │      ▼                  ▼                 │
                    │  Ollama (LLM)      Neo4j (vector store)   │
                    │  port 11434        ports 7474 / 7687      │
                    │  tinyllama /       embeddings via         │
                    │  llama2 /          sentence-transformers  │
                    │  neural-chat                              │
                    └─────────────────────────────────────────┘
                         all running under Docker Compose
```

## Tech stack

| Layer            | Technology                                  |
|------------------|---------------------------------------------|
| UI               | Streamlit                                   |
| Orchestration    | LangChain                                   |
| LLM serving      | Ollama (tinyllama, llama2, neural-chat)     |
| Vector store     | Neo4j 5.11                                  |
| Embeddings       | sentence-transformers                       |
| Containerization | Docker + Docker Compose                     |
| Host             | Ubuntu 22.04 LTS VM (cloud)                 |

## What it does

1. Upload a PDF through the web UI.
2. The document is chunked, embedded with sentence-transformers, and stored as
   vectors in Neo4j.
3. A natural-language question is embedded and matched against those vectors.
4. The most relevant chunks are passed to the LLM (served by Ollama) as context.
5. The model returns an answer grounded in the document.

## Model comparison

The same pipeline was run against three models to observe the trade-off between
response quality, latency, and resource use:

| Model        | Size    | Notes                                            |
|--------------|---------|--------------------------------------------------|
| tinyllama    | ~640 MB | Fast to load, lightweight, lower coherence       |
| llama2       | ~3.8 GB | Noticeably more coherent, slower responses       |
| neural-chat  | ~4.1 GB | Most polished, conversational answers            |

Switching models is a one-line change in `.env` (`LLM=<model>`) plus a restart.

## Running it

Prerequisites: a Linux VM with Docker and Docker Compose installed, and inbound
access to ports 8000, 11434, 7474, and 7687.

```bash
# 1. Clone and enter the project
git clone <this-repo-url>
cd genai-pdf-chat-docker

# 2. Create your environment file from the template
cp .env.example .env
nano .env          # fill in your values (see Configuration below)

# 3. Bring the stack up (first build takes several minutes)
docker compose up -d

# 4. Pull a model into the Ollama container
docker exec -it $(docker ps -qf "name=ollama") ollama pull tinyllama

# 5. Open the app
#    http://<your-vm-ip>:8000
```

To swap models, edit `LLM=` in `.env`, run `docker compose up -d`, and pull the
new model with the `ollama pull` command above.

## Configuration

See [`.env.example`](.env.example). Key values:

- `LLM` — the model Ollama serves (`tinyllama`, `llama2`, or `neural-chat`).
- `EMBEDDING_MODEL` — `sentence_transformer`.
- `NEO4J_URI` — points at the **local** Neo4j container: `neo4j://database:7687`.
- `NEO4J_USERNAME` / `NEO4J_PASSWORD` — credentials for the local Neo4j instance.
- `OLLAMA_BASE_URL` — `http://ollama:11434`.

## Engineering notes / troubleshooting

Getting a multi-service AI pipeline running end to end meant debugging across
every layer of the stack. The full write-up is in
[`docs/TROUBLESHOOTING.md`](docs/TROUBLESHOOTING.md). Highlights:

- **Neo4j version mismatch.** A managed cloud Neo4j instance had been upgraded
  past the `db.index.vector.createNodeIndex` procedure the app's LangChain
  version expected. Resolved by repointing the app to the version-matched
  Neo4j 5.11 container that Compose already runs.
- **Ollama segmentation fault.** The model server crashed during warm-up inside
  an AMX CPU-acceleration code path in a bleeding-edge build. Resolved by
  pinning the Ollama image to a known-stable version instead of `latest`.
- **Operational hygiene.** Config typos, editor mix-ups, and SSH timeouts mid
  download (solved with `tmux`).

## What I learned

A RAG pipeline is only as strong as its weakest layer. The tutorial gets you
most of the way; the real skill is reading logs, isolating the failing layer,
and telling the difference between a memory issue, a version mismatch, and a
corrupted file.

## License

MIT — see [LICENSE](LICENSE).
