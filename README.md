# ameth — Vector Database from Scratch in C++

A vector database built from the ground up in C++ with a web UI, written to understand how production AI retrieval systems actually work under the hood — by implementing the core algorithms myself.

Includes a **RAG pipeline** powered by a local LLM via Ollama, so the whole stack — embedding, indexing, retrieval, and generation — runs locally.

> Built as an project to learn and understand how production vector databases like Pinecone, Weaviate, and Chroma actually work under the hood, and implement from first principles.

---

## What It Does

| Feature | Description |
|---|---|
| **3 Search Algorithms** | HNSW, KD-Tree, and Brute Force — all three run in parallel so you can benchmark them |
| **3 Distance Metrics** | Cosine similarity, Euclidean, Manhattan |
| **2D PCA Scatter Plot** | Live visualization of the semantic space as vectors are added |
| **Real Document Embedding** | Paste any text → Ollama embeds it with `nomic-embed-text` (768D) |
| **RAG Pipeline** | Question → HNSW retrieval → local LLM answer, fully offline |
| **REST API** | CRUD endpoints: insert, delete, search, benchmark, hnsw-info |

---

## How the Stack Works

```
Your Text
    │
    ▼
Ollama (nomic-embed-text)     ← text → 768-dimensional vector
    │
    ▼
HNSW Index (C++)              ← multilayer graph index
    │
    ▼
Semantic Search               ← nearest-neighbor retrieval
    │
    ▼
Ollama (llama3.2)             ← generates an answer from retrieved context
    │
    ▼
Answer
```

---

## Algorithms

### HNSW (Hierarchical Navigable Small World)

The same algorithm used by Pinecone, Weaviate, Chroma, and Milvus. Builds a multilayer graph where each layer is progressively sparser — searches start at the top and zoom in, giving O(log N) complexity vs O(N) for brute force.

Each node is assigned a random max layer on insert. From its layer down to 0, a beam search finds the nearest neighbors and connects them bidirectionally. At search time, the upper layers act as a highway to the right neighborhood, and layer 0 does the fine-grained scan.

### KD-Tree

Binary space partitioning — splits space along one dimension at a time and prunes subtrees during search when they can't possibly beat the current best candidate. Works well in low dimensions but degrades badly past ~20D due to the curse of dimensionality.

### Why HNSW wins at high dimensions

KD-Tree pruning relies on axis-aligned distance bounds. In high dimensions, almost everything sits near the surface of the hypersphere — nothing gets pruned. HNSW's graph-based traversal doesn't have this problem, which is why it's the industry standard for 768D+ embeddings.

---

## Project Structure

```
ameth/
├── main.cpp        ← C++ backend: HNSW, KD-Tree, BruteForce, REST API, RAG
├── httplib.h       ← Single-header HTTP server (cpp-httplib)
├── index.html      ← Frontend: PCA scatter plot, chat UI, benchmark view
└── README.md
```

### Architecture

```
BruteForce      O(N·d)      Exact, baseline
KDTree          O(log N)    Exact, axis-aligned partitioning
HNSW            O(log N)    Approximate, multilayer small-world graph

VectorDB        Unified interface over all 3 (16D demo vectors)
DocumentDB      HNSW-only index for real Ollama embeddings (768D)
OllamaClient    HTTP client → /api/embeddings + /api/generate
```

---

## Setup (Windows)

**Prerequisites:** MSYS2 (g++ compiler), Git, Ollama

```bash
# Pull models
ollama pull nomic-embed-text
ollama pull llama3.2

# Compile
g++ -std=c++17 -O2 main.cpp -o db -lws2_32

# Run
ollama serve        # Terminal 1 (skip if already in system tray)
./db                # Terminal 2
```

Open `http://localhost:8080`.

---

## REST API

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/search?v=...&k=5&metric=cosine&algo=hnsw` | K-NN search |
| `POST` | `/insert` | Insert a vector |
| `DELETE` | `/delete/:id` | Delete by ID |
| `GET` | `/benchmark?v=...&k=5&metric=cosine` | Compare all 3 algorithms |
| `GET` | `/hnsw-info` | Graph structure and layer stats |
| `POST` | `/doc/insert` | Embed and store a document |
| `GET` | `/doc/list` | List stored documents |
| `POST` | `/doc/ask` | RAG: retrieve + generate |
| `GET` | `/status` | Ollama status |

---

## License

MIT
