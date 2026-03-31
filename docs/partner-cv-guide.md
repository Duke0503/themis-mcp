# Partner CV Guide — Themis Project

**Mục đích**: Giúp partner tối đa hóa giá trị CV từ dự án này khi apply job.

---

## Kỹ thuật nên nắm vững (và có thể giải thích trong phỏng vấn)

### Must-know (bắt buộc giải thích được)

**1. LangChain — tại sao dùng và dùng phần nào**

Interviewer sẽ hỏi: *"Tại sao dùng LangChain thay vì tự implement?"*

Trả lời: *"LangChain cung cấp MultiQueryRetriever — tự động generate 3 query variants rồi merge results. Với legal text, một câu hỏi có thể diễn đạt nhiều cách khác nhau. Multi-query giúp tăng recall. Ngoài ra ContextualCompressionRetriever wrap cross-encoder reranker vào retriever chain rất clean."*

**2. Hybrid Search — giải thích RRF**

Interviewer sẽ hỏi: *"Hybrid search là gì, tại sao tốt hơn pure vector search?"*

Trả lời: *"Vector search giỏi semantic similarity nhưng miss exact keyword matches — tên luật, số điều khoản, mã văn bản. BM25 giỏi exact match nhưng miss paraphrase. Hybrid kết hợp cả hai qua Reciprocal Rank Fusion: score = sum(1/(rank_i + 60)) cho mỗi result set. Trong production, hybrid thường tốt hơn pure vector 10–20% precision."*

**3. Cross-encoder Reranking — giải thích tại sao cần 2 bước**

Interviewer sẽ hỏi: *"Tại sao cần reranking, embedding không đủ à?"*

Trả lời: *"Bi-encoder (embedding) chạy nhanh nhưng encode query và document độc lập — không capture interaction giữa chúng. Cross-encoder encode cặp (query, document) cùng lúc nên accurate hơn nhiều, nhưng O(n) với n candidates. Vì vậy dùng bi-encoder để retrieve 20 candidates, rồi cross-encoder để rerank top 5. Pattern này là industry standard."*

**4. Chunking strategy cho legal documents**

Interviewer sẽ hỏi: *"Chunking strategy của bạn là gì?"*

Trả lời: *"Legal documents có structure đặc biệt — Vietnamese 'Điều X.' và English 'Article X'. Detect pattern này và chunk theo từng điều luật, giữ nguyên legal provision hoàn chỉnh. Fallback về heading-based rồi paragraph-based. Chunk size 512 tokens với 50 token overlap. Quan trọng nhất là không cắt giữa điều luật — context loss nghiêm trọng với legal text."*

**5. Multilingual embeddings**

Interviewer sẽ hỏi: *"Tại sao chọn multilingual-e5-small?"*

Trả lời: *"Dự án cần search cả Vietnamese và English documents với cùng một query. Multilingual-e5 được train trên 100+ ngôn ngữ, 384 dimensions, chạy local không cần API key. Quan trọng: e5 yêu cầu prefix 'query: ' cho queries và 'passage: ' cho documents — bỏ qua prefix này làm giảm quality đáng kể."*

---

### Good-to-know (nếu hỏi sâu hơn)

**RAG Evaluation Metrics**
- Precision@K: trong K results, bao nhiêu % relevant
- MRR (Mean Reciprocal Rank): relevant result xuất hiện ở rank bao nhiêu
- RAGAS framework: faithfulness, answer relevancy, context recall

**MCP Protocol**
- stdio transport cho local, SSE cho remote
- Tool calling pattern: AI agent tự quyết định khi nào gọi tool
- Khác chatbot: không có UI, agent-driven

**Qdrant internals**
- HNSW index cho dense vector
- Sparse vector cho keyword (BM25/SPLADE)
- Payload filtering: WHERE clause cho vector search

---

## CV bullet points (copy-paste và điền số liệu thật)

```
Themis — Legal Intelligence Platform | AI/ML Engineer | 2026
GitHub: github.com/yourname/themis

• Designed and implemented hybrid RAG pipeline using LangChain:
  MultiQueryRetriever + Qdrant hybrid search (BM25 + dense) + cross-encoder reranking
  → Achieved [X]% Precision@5 on multilingual Vietnamese/English legal queries

• Built legal-aware document chunking system supporting 7 formats (PDF, DOCX, MD, OCR):
  Vietnamese "Điều X" and English "Article X" article detection preserves complete legal provisions

• Integrated multilingual-e5-small embeddings (HuggingFace, local inference):
  100+ language support, no API key required, <30s cold start on CPU

• Implemented MCP server (Model Context Protocol) enabling AI agents to automatically
  retrieve legal knowledge — integrated with Claude Code, Cursor, GitHub Copilot

• Achieved [X]ms p95 search latency with Qdrant HNSW index and result caching

• Built Google Drive connector with OAuth 2.0 and incremental sync:
  Only re-indexes modified files, [X]% faster than full reindex

Tech: Python, LangChain, HuggingFace Transformers, Qdrant, FastAPI, Docker,
      MCP Protocol, Google Drive API, cross-encoder, multilingual-e5
```

---

## Keywords để thêm vào LinkedIn / CV skill section

```
LangChain · LlamaIndex · RAG (Retrieval-Augmented Generation) · Vector Database
Qdrant · FAISS · Pinecone · HuggingFace Transformers · Sentence Transformers
Hybrid Search · BM25 · Cross-encoder Reranking · Reciprocal Rank Fusion
MCP (Model Context Protocol) · AI Agents · Agentic AI
FastAPI · Python · Docker · Google Drive API · OAuth 2.0
Multilingual NLP · Vietnamese NLP · Document Understanding
RAG Evaluation · Precision@K · MRR · RAGAS
```

---

## Jobs phù hợp sau project này

| Role | Company types | Keywords match |
|---|---|---|
| AI/ML Engineer | AI startups, LegalTech, FinTech | LangChain, RAG, embeddings |
| MLOps Engineer | Any company with AI products | Qdrant, Docker, FastAPI, evaluation |
| AI Platform Engineer | Big tech, consultancy | MCP, agents, retrieval systems |
| LLM Engineer | AI-first companies | RAG, LangChain, HuggingFace |
| Backend Engineer (AI) | Product companies | FastAPI, Python, vector DB |

---

## Cách giải thích dự án trong 1 phút (elevator pitch)

> "Tôi build Themis — một legal intelligence platform dành cho startups Việt Nam. Vấn đề là AI models như ChatGPT hallucinate khi hỏi về luật Việt Nam vì chúng không có access đến primary sources. Themis index toàn bộ văn bản pháp luật — từ Luật PCRT đến GDPR — vào Qdrant vector database, sau đó expose qua MCP protocol để Claude, Cursor và các AI agent khác tự động tra cứu khi cần. Điểm kỹ thuật interesting nhất là hybrid search pipeline: BM25 cho exact keyword match, dense vector cho semantic similarity, merge qua Reciprocal Rank Fusion, rồi rerank bằng cross-encoder. Precision@5 đạt [X]% trên Vietnamese legal queries."
