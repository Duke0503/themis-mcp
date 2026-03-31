# Themis — Architecture

---

## System Overview

Themis có 2 service tách biệt rõ ràng:

```
┌─────────────────────────────────────┐
│          MCP Server (Node.js)       │  ← Duke owns this
│  Thin wrapper, protocol translation │
│  stdio / SSE transport              │
└──────────────┬──────────────────────┘
               │ HTTP (internal Docker network)
┌──────────────▼──────────────────────┐
│      AI Pipeline (Python)           │  ← Partner owns this
│  LangChain · HuggingFace · Qdrant  │
│  All AI/ML logic lives here         │
└─────────────────────────────────────┘
```

**Lý do tách 2 service:**
- MCP protocol = Node.js (SDK tốt nhất là JS)
- AI/ML = Python (LangChain, HuggingFace mature nhất ở Python)
- Tách ra → có thể scale độc lập
- Tách ra → Partner và Duke làm song song không conflict

---

## Data Flow: Indexing

```
Source (file / Google Drive / Notion)
         │
         ▼
    Connector
    (download + watch)
         │ raw bytes
         ▼
    Parser
    (PDF → PyMuPDF, DOCX → python-docx, IMG → Tesseract)
         │ plain text + metadata
         ▼
    Chunker
    (LangChain RecursiveCharacterTextSplitter
     + legal article regex: Điều X / Article X)
         │ chunks[]
         ▼
    Embedder
    (multilingual-e5-small, prefix: "passage: ")
         │ dense vectors[]
         ▼
    Sparse Encoder
    (BM25 / SPLADE for keyword index)
         │ sparse vectors[]
         ▼
    Qdrant Upsert
    (dense + sparse + metadata payload)
```

---

## Data Flow: Search Query

```
MCP Client (Claude) calls search("AML requirements Vietnam")
         │
         ▼
MCP Server (Node.js) → POST /search to Python API
         │
         ▼
MultiQueryRetriever (LangChain)
  → Generate 3 query variants via LLM
  → "KYC regulations Vietnam fintech"
  → "PCRT 2022 requirements"
  → "AML compliance Vietnamese crypto exchange"
         │
         ▼
Qdrant Hybrid Search (per query variant)
  → Dense: multilingual-e5 vector similarity
  → Sparse: BM25 keyword matching
  → Merge: Reciprocal Rank Fusion (k=60)
         │ 20 candidates
         ▼
Cross-encoder Reranker
  (ms-marco-MiniLM-L-6-v2)
  → Score each (query, chunk) pair
  → Return top 5
         │ 5 results
         ▼
Confidence Scoring
  → similarity ≥ 0.7 → HIGH
  → similarity ≥ 0.5 → MEDIUM
  → similarity < 0.5 → LOW
         │
         ▼
Response with citations
  { text, source, section, article, similarity, confidence }
```

---

## Module Responsibilities

| Module | Owner | Depends on |
|---|---|---|
| `mcp-server/server.js` | Duke | `@modelcontextprotocol/sdk`, Pipeline API |
| `pipeline/main.py` | Partner | FastAPI, all pipeline modules |
| `pipeline/searcher.py` | Partner | LangChain, Qdrant, cross-encoder |
| `pipeline/chunker.py` | Partner | LangChain text splitters |
| `pipeline/embeddings.py` | Partner | langchain-huggingface, HF Transformers |
| `pipeline/indexer.py` | Partner | Parsers, chunker, embeddings, Qdrant |
| `pipeline/connectors/google_drive.py` | Duke | Google API client |
| `pipeline/parsers/*.py` | Partner | PyMuPDF, python-docx, Tesseract |

---

## Docker Compose

```yaml
services:
  qdrant:
    image: qdrant/qdrant:latest
    ports: ["6333:6333"]
    volumes: ["qdrant_data:/qdrant/storage"]

  pipeline:
    build: ./pipeline
    ports: ["8000:8000"]
    environment:
      - QDRANT_URL=http://qdrant:6333
      - HF_HOME=/models
    volumes:
      - ./knowledge:/app/knowledge
      - ./models:/models
    depends_on: [qdrant]

  mcp-server:
    build: ./mcp-server
    environment:
      - THEMIS_API=http://pipeline:8000
    depends_on: [pipeline]
    stdin_open: true
    tty: true

volumes:
  qdrant_data:
```

---

## Key Design Decisions

### 1. Python for AI, Node.js for MCP
LangChain Python > LangChain JS (maturity, features). MCP SDK Node.js > Python SDK (more complete). Dùng đúng tool cho đúng job.

### 2. Qdrant thay vì vectra
vectra = JSON file, không có hybrid search native. Qdrant = production vector DB, sparse+dense built-in, Docker-native.

### 3. LangChain làm orchestration layer
Không reinvent wheel. MultiQueryRetriever, ContextualCompressionRetriever, ConversationalRetrievalChain là battle-tested patterns. Partner có thể focus vào domain-specific tuning.

### 4. Cross-encoder reranking là optional layer
Precision +15–30% nhưng latency +200–500ms. Feature flag `ENABLE_RERANKING=true/false`. Default true, có thể tắt nếu cần speed.

### 5. MCP server là thin proxy
Không có business logic ở Node.js layer. Chỉ validate input (Zod), forward đến Python API, format response theo MCP spec. Toàn bộ AI logic ở Python.
