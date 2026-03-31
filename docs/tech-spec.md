# Themis — Technical Specification

---

## 1. System Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    MCP Clients                           │
│         Claude Code · Cursor · GitHub Copilot           │
└────────────────────────┬─────────────────────────────────┘
                         │ MCP Protocol (stdio / SSE)
┌────────────────────────▼─────────────────────────────────┐
│                  MCP Server (Node.js)                    │
│   /mcp-server                                            │
│   Tools: search · list_sources · get_document · refresh  │
│   SDK: @modelcontextprotocol/sdk                         │
└────────────────────────┬─────────────────────────────────┘
                         │ HTTP REST (internal)
┌────────────────────────▼─────────────────────────────────┐
│              AI Pipeline API (Python / FastAPI)          │
│   /pipeline                                              │
│                                                          │
│  ┌──────────────┐  ┌───────────────┐  ┌───────────────┐ │
│  │  LangChain   │  │  Connectors   │  │   Indexer     │ │
│  │  Retrieval   │  │  Google Drive │  │   Pipeline    │ │
│  │  Chain       │  │  Notion       │  │   File Watch  │ │
│  └──────┬───────┘  └───────┬───────┘  └───────┬───────┘ │
│         │                  │                  │          │
│  ┌──────▼──────────────────▼──────────────────▼───────┐  │
│  │              Qdrant Vector Store                    │  │
│  │   Dense vectors + BM25 sparse + metadata filter    │  │
│  └─────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Tech Stack

### AI Pipeline (Python)

| Component | Library | Version | Role |
|---|---|---|---|
| Orchestration | **LangChain** | ^0.3.x | Document loading, retrieval chains, agents |
| Embeddings | **langchain-huggingface** | ^0.1.x | HuggingFace embedding wrapper |
| Embedding model | **multilingual-e5-small** | via HF | 384-dim, 100+ languages, local |
| Vector store | **langchain-qdrant** | ^0.2.x | LangChain ↔ Qdrant integration |
| Reranking | **sentence-transformers** | ^3.x | cross-encoder/ms-marco-MiniLM-L-6-v2 |
| PDF parsing | **PyMuPDF (fitz)** | ^1.24 | Better than pdfminer for tables |
| DOCX parsing | **python-docx** | ^1.x | Word document extraction |
| OCR | **pytesseract** | ^0.3.x | Image → text (Vietnamese + English) |
| API server | **FastAPI** | ^0.115 | Internal HTTP API |
| Task queue | **Celery** | ^5.x | Background indexing jobs |

### MCP Server (Node.js)

| Component | Library | Version | Role |
|---|---|---|---|
| MCP protocol | **@modelcontextprotocol/sdk** | ^1.28 | MCP server implementation |
| HTTP client | **axios** | ^1.x | Calls Python pipeline API |
| Validation | **zod** | ^3.x | Input schema validation |

### Infrastructure

| Component | Technology | Role |
|---|---|---|
| Vector DB | **Qdrant** | Stores embeddings + metadata, hybrid search |
| Containerization | **Docker Compose** | Wires all services together |
| Connector auth | **Google OAuth 2.0** | Google Drive access |

---

## 3. LangChain Integration (Key Design)

### 3.1 Why LangChain?

LangChain provides battle-tested patterns for:
- **MultiQueryRetriever**: generates 3 query variants → merges results → eliminates single-query bias
- **ContextualCompressionRetriever**: wraps any retriever with a cross-encoder reranker
- **ConversationalRetrievalChain**: conversation memory + retrieval in one chain
- **Document Loaders**: standardized interface for 50+ source types

### 3.2 Retrieval Chain Architecture

```python
from langchain.retrievers import MultiQueryRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain.retrievers import ContextualCompressionRetriever
from langchain_qdrant import QdrantVectorStore
from langchain_huggingface import HuggingFaceEmbeddings

# Base retriever: Qdrant hybrid search
base_retriever = QdrantVectorStore(...).as_retriever(
    search_type="mmr",    # Maximal Marginal Relevance — diversity
    search_kwargs={"k": 20, "fetch_k": 50}
)

# Layer 1: Multi-query expansion
multi_query_retriever = MultiQueryRetriever.from_llm(
    retriever=base_retriever,
    llm=llm,              # Generates 3 query variants
)

# Layer 2: Cross-encoder reranking
reranker = CrossEncoderReranker(
    model=CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2"),
    top_n=5
)
compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=multi_query_retriever
)

# Final chain
chain = ConversationalRetrievalChain.from_llm(
    llm=llm,
    retriever=compression_retriever,
    return_source_documents=True,
    verbose=True
)
```

### 3.3 Document Ingestion with LangChain

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.document_loaders import (
    PyMuPDFLoader,
    UnstructuredWordDocumentLoader,
    UnstructuredMarkdownLoader
)

# Custom separators for Vietnamese/English legal docs
legal_splitter = RecursiveCharacterTextSplitter(
    separators=[
        "\nĐiều \d+\.",      # Vietnamese article
        "\nArticle \d+",     # English article
        "\n## ",             # Markdown h2
        "\n### ",            # Markdown h3
        "\n\n",              # Paragraph
        "\n",
        " "
    ],
    chunk_size=512,
    chunk_overlap=50,
    length_function=count_tokens,  # Token-based, not char-based
)
```

---

## 4. Hybrid Search Implementation

### 4.1 Qdrant Hybrid Search

Qdrant natively supports sparse + dense hybrid search via `query_points`:

```python
from qdrant_client.models import SparseVector, NamedSparseVector

# Dense query vector (multilingual-e5)
dense_vector = embedder.embed_query(f"query: {query}")

# Sparse query vector (BM25 via SPLADE or BM42)
sparse_vector = sparse_encoder.encode(query)

results = qdrant_client.query_points(
    collection_name="themis",
    query=dense_vector,
    using="dense",
    sparse_vector=NamedSparseVector(
        name="sparse",
        vector=SparseVector(indices=sparse_vector.indices, values=sparse_vector.values)
    ),
    limit=20
)
```

### 4.2 RRF Merge (if manual)

```python
def reciprocal_rank_fusion(results_list: list[list], k: int = 60) -> list:
    scores = {}
    for results in results_list:
        for rank, doc in enumerate(results):
            doc_id = doc.metadata["chunk_id"]
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (rank + k)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

---

## 5. Indexing Pipeline

```
Source File
    │
    ▼
┌─────────────┐
│   Parser    │  PyMuPDF / python-docx / markdown / tesseract
└──────┬──────┘
       │ raw text + metadata
       ▼
┌─────────────┐
│   Chunker   │  LangChain RecursiveCharacterTextSplitter
│             │  + legal article regex detection
└──────┬──────┘
       │ chunks[]
       ▼
┌─────────────┐
│  Embedder   │  HuggingFaceEmbeddings("intfloat/multilingual-e5-small")
│             │  prefix: "passage: " + chunk.text
└──────┬──────┘
       │ vectors[]
       ▼
┌─────────────┐
│   Filter    │  Discard chunks with similarity < 0.3 vs PROJECT_CONTEXT
└──────┬──────┘
       │ filtered vectors[]
       ▼
┌─────────────┐
│   Upsert    │  Qdrant: dense vector + sparse vector + metadata
└─────────────┘
```

### Incremental Indexing

```python
import hashlib

def file_hash(path: str) -> str:
    return hashlib.sha256(open(path, "rb").read()).hexdigest()

def needs_reindex(path: str, index_cache: dict) -> bool:
    current_hash = file_hash(path)
    return index_cache.get(path) != current_hash
```

---

## 6. MCP Server (Node.js)

### 6.1 Tools

```typescript
// search
{
  name: "search",
  description: "Search the legal knowledge base with semantic + keyword hybrid search",
  inputSchema: {
    query: z.string(),
    category: z.enum(["vietnam","international","gaming-tos","escrow-regs","tax"]).optional(),
    limit: z.number().min(1).max(20).default(5)
  }
}

// list_sources
{
  name: "list_sources",
  description: "List all indexed documents",
  inputSchema: {
    category: z.string().optional()
  }
}

// get_document
{
  name: "get_document",
  description: "Get full content of a document by path",
  inputSchema: {
    path: z.string()
  }
}

// refresh
{
  name: "refresh",
  description: "Trigger re-sync of a connector",
  inputSchema: {
    connector: z.enum(["google-drive", "notion", "local"]),
    category: z.string().optional()
  }
}
```

### 6.2 Transport

```
Local self-host: stdio (stdin/stdout)
Cloud endpoint:  SSE (Server-Sent Events) over HTTPS
```

---

## 7. Google Drive Connector

```python
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

class GoogleDriveConnector:
    SCOPES = ["https://www.googleapis.com/auth/drive.readonly"]

    def list_files(self, folder_id: str) -> list[DriveFile]:
        """List all files in folder, respecting permissions"""

    def download_file(self, file_id: str) -> bytes:
        """Download file content"""

    def watch_folder(self, folder_id: str, webhook_url: str):
        """Setup Drive push notification for changes"""

    def sync(self, folder_id: str, category: str):
        """Full sync: download → parse → chunk → embed → upsert"""

    def incremental_sync(self, folder_id: str, since: datetime):
        """Only sync files modified since last sync"""
```

---

## 8. API Endpoints (Python FastAPI)

```
POST /search
  body: { query, category?, limit? }
  returns: { results[], latency_ms }

GET  /sources?category=
  returns: { sources[], total_documents, total_chunks }

GET  /document?path=
  returns: { content, metadata }

POST /index
  body: { path?, category?, force? }
  returns: { job_id }

GET  /job/{job_id}
  returns: { status, progress, errors[] }

POST /connectors/google-drive/sync
  body: { folder_id, category }
  returns: { job_id }
```

---

## 9. Data Model

### Qdrant Point (per chunk)

```python
{
  "id": "uuid",
  "vector": {
    "dense": [0.023, -0.112, ...],   # 384 dims, multilingual-e5
    "sparse": {"indices": [...], "values": [...]}  # BM25/SPLADE
  },
  "payload": {
    "text": "Điều 1. Phạm vi áp dụng...",
    "source": "vietnam/pcrt-2022.pdf",
    "section": "Điều 1",
    "category": "vietnam",
    "title": "Luật Phòng chống rửa tiền 2022",
    "jurisdiction": "Vietnam",
    "priority": 1,
    "effective_date": "2022-03-01",
    "tags": ["AML", "KYC"],
    "file_hash": "sha256:abc123...",
    "indexed_at": "2026-03-31T00:00:00Z",
    "chunk_index": 3,
    "total_chunks": 47
  }
}
```

---

## 10. Performance Targets

| Operation | Target | Strategy |
|---|---|---|
| Cold start | < 5s | Lazy model load, Qdrant pre-warmed |
| First query (model load) | < 30s | One-time, cached after |
| Search (after warmup) | < 2s | Qdrant HNSW index, result cache |
| Index 100 docs | < 10 min | Batch embedding, async upsert |
| Google Drive sync (100 files) | < 5 min | Parallel download + index |

---

## 11. Project Structure

```
themis/
├── docker-compose.yml
├── README.md
├── pipeline/                    # Python AI pipeline
│   ├── main.py                  # FastAPI app
│   ├── cli.py                   # CLI commands
│   ├── requirements.txt
│   ├── config.py                # All constants
│   ├── indexer.py               # Indexing pipeline
│   ├── searcher.py              # Hybrid search + reranking
│   ├── chunker.py               # Legal-aware chunking
│   ├── embeddings.py            # HuggingFace wrapper
│   ├── connectors/
│   │   ├── google_drive.py
│   │   ├── notion.py
│   │   └── local_files.py
│   └── parsers/
│       ├── pdf.py               # PyMuPDF
│       ├── docx.py              # python-docx
│       ├── markdown.py          # YAML frontmatter
│       ├── image.py             # Tesseract OCR
│       └── registry.py
├── mcp-server/                  # Node.js MCP server
│   ├── server.js
│   ├── package.json
│   └── tools/
│       ├── search.js
│       ├── list_sources.js
│       ├── get_document.js
│       └── refresh.js
├── knowledge/
│   ├── vietnam/
│   ├── international/
│   ├── gaming-tos/
│   ├── escrow-regs/
│   └── tax/
└── docs/
    ├── prd.md
    ├── tech-spec.md
    ├── architecture.md
    └── roadmap.md
```
