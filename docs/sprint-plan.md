# Themis — Sprint Plan (4 tuần)

**Duke**: SE/Architect — owns MCP server, Google Drive, deploy
**Partner**: AI Specialist — owns toàn bộ AI pipeline (Python/LangChain)

---

## Week 1 — AI Pipeline Foundation

### Partner (Python)
- [ ] Setup Python environment, `requirements.txt`, Docker cho pipeline service
- [ ] Implement tất cả parsers: `pdf.py`, `docx.py`, `markdown.py`, `image.py`, `registry.py`
- [ ] Implement `chunker.py` — LangChain RecursiveCharacterTextSplitter + legal article regex
- [ ] Implement `embeddings.py` — HuggingFace multilingual-e5-small wrapper
- [ ] Unit tests cho chunker với 10+ edge cases (Điều luật tiếng Việt, Article English, paragraph thường)

### Duke (Node.js + Infra)
- [ ] Setup GitHub repo, CI (GitHub Actions)
- [ ] Setup Docker Compose skeleton (Qdrant + pipeline + mcp-server)
- [ ] Viết `docs/architecture.md` — interface contracts giữa các modules
- [ ] Define FastAPI endpoints spec (Partner sẽ implement)

**Checkpoint cuối tuần 1:**
```bash
python -c "from pipeline.chunker import chunk; print(chunk('Điều 1. Test\n\nĐiều 2. Test2', 'law.pdf'))"
# → [{"text": "Điều 1. Test", "section": "Điều 1"}, ...]
```

---

## Week 2 — Indexer + Search + CLI

### Partner (Python)
- [ ] Setup Qdrant collection (dense + sparse vectors)
- [ ] Implement `indexer.py` — full pipeline: walk files → parse → chunk → embed → upsert
- [ ] Implement `searcher.py` — hybrid search (BM25 + Qdrant) + RRF merge
- [ ] Implement `main.py` — FastAPI: `/search`, `/sources`, `/index`, `/document`
- [ ] Implement `cli.py` — index, search, status commands
- [ ] Incremental indexing (SHA256 file hash cache)

### Duke
- [ ] Test Qdrant connection, verify collection setup
- [ ] Integration test: index 10 real legal PDFs, verify chunks in Qdrant
- [ ] Review Partner's searcher.py — check RRF implementation
- [ ] Begin Google Drive OAuth setup (Google Cloud Console)

**Checkpoint cuối tuần 2:**
```bash
python pipeline/cli.py index knowledge/
python pipeline/cli.py search "GDPR data retention"
# → results với source citations, confidence scores
```

---

## Week 3 — Reranking + MCP Server + Google Drive

### Partner (Python)
- [ ] Implement cross-encoder reranking (`sentence-transformers`, ms-marco-MiniLM)
- [ ] Implement `MultiQueryRetriever` (LangChain) vào search pipeline
- [ ] Implement `ContextualCompressionRetriever` wrapping reranker
- [ ] **Evaluate và tune**: chạy 20+ test queries, measure Precision@5, MRR
- [ ] Document findings: baseline vs hybrid vs reranking

### Duke (Node.js)
- [ ] Implement `mcp-server/server.js` — MCP tools: search, list_sources, get_document, refresh
- [ ] Implement Google Drive connector (`connectors/google_drive.py`) — OAuth + download + sync
- [ ] Test end-to-end: Claude Code → MCP → Python API → Qdrant → response

**Checkpoint cuối tuần 3:**
```
Claude Code: "Search for KYC requirements in Vietnam"
→ cites specific articles from vietnamese legal docs ✅
```

**Partner deliverable tuần 3:**
```
Retrieval Quality Report:
- Baseline (dense only):   Precision@5 = ?%
- Hybrid (BM25 + dense):   Precision@5 = ?%
- + MultiQuery:            Precision@5 = ?%
- + Reranking:             Precision@5 = ?%
- Latency per step: ?ms
```

---

## Week 4 — Polish + Deploy + Docs

### Partner (Python)
- [ ] Fix tất cả bugs từ integration testing
- [ ] Optimize: batch embedding, async Qdrant upsert
- [ ] Viết `docs/tech-spec.md` (AI pipeline sections)
- [ ] Viết knowledge base guide (`docs/knowledge-base-guide.md`)
- [ ] Populate `knowledge/` với ít nhất 10 real legal documents (PDF/MD)
- [ ] Pre-build Qdrant index cho demo

### Duke (Node.js + Infra)
- [ ] Deploy cloud (Railway hoặc Fly.io)
- [ ] Performance test: cold start < 5s, search < 2s
- [ ] Run acceptance criteria (8 tests)
- [ ] Viết README (quick start, MCP config snippets)
- [ ] GitHub repo public, clean history

**End of Week 4 — Done:**
```
✅ docker compose up → tất cả services chạy
✅ python cli.py search "query" → < 2s
✅ Claude Code/Cursor cite legal sources tự động
✅ GitHub public repo với pre-built index
✅ Demo video 2 phút
```
