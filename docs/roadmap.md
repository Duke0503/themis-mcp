# Themis — Roadmap & Expansion Vision

---

## Mục tiêu cuối của dự án

### Ngắn hạn (6 tháng)
> **Trở thành legal knowledge layer chuẩn cho AI agent trong hệ sinh thái startup Việt Nam.**

Mọi developer dùng Claude Code, Cursor, hay Copilot khi làm việc với dự án có liên quan đến pháp lý ở Việt Nam đều có thể hỏi và nhận câu trả lời có trích dẫn nguồn — không hallucinate, không cần rời khỏi editor.

### Trung hạn (12 tháng)
> **Mở rộng ra Southeast Asia — Thái Lan, Singapore, Indonesia, Philippines.**

Mỗi quốc gia có knowledge base riêng, Themis route query đến đúng jurisdiction.

### Dài hạn (2–3 năm)
> **Themis trở thành infrastructure layer cho legal AI applications.**

Bất kỳ startup nào muốn build legal AI product (chatbot luật, compliance checker, contract reviewer) đều dùng Themis như backend RAG engine — giống như Stripe cho payments, hay Twilio cho SMS.

---

## Phase 1 — Foundation (Month 1) ✅ Current

**Focus**: Core RAG pipeline hoạt động tốt cho legal documents.

- [x] Hybrid search (BM25 + dense vector)
- [x] Legal-aware chunking (Vietnamese + English article detection)
- [x] MCP server (stdio)
- [x] PDF, DOCX, Markdown parsing
- [x] Local embeddings (multilingual-e5-small)
- [x] Docker self-host

**Success metric**: Precision@5 ≥ 70% trên Vietnamese legal queries.

---

## Phase 2 — Connectors (Month 2)

**Focus**: Documents đến từ nơi người ta thực sự lưu trữ chúng.

- [ ] Google Drive connector (OAuth 2.0, auto-sync)
- [ ] Notion connector (pages + databases)
- [ ] Cross-encoder reranking
- [ ] MultiQueryRetriever (LangChain)
- [ ] Incremental sync (chỉ re-index file đã thay đổi)
- [ ] Cloud MCP endpoint (SSE transport)

**Success metric**: Team có thể connect Google Drive → index → query trong < 10 phút.

---

## Phase 3 — Intelligence (Month 3–4)

**Focus**: Câu trả lời thông minh hơn, không chỉ retrieval đơn giản.

- [ ] **Agentic RAG**: Multi-step retrieval — decompose complex queries thành sub-queries, route đến đúng category, synthesize kết quả
- [ ] **Conflict detection**: Khi 2 documents có quy định mâu thuẫn nhau (vd: luật cũ vs luật mới), flag rõ ràng
- [ ] **Document freshness**: Track effective_date, flag documents sắp hết hiệu lực
- [ ] **Citation quality**: Link trực tiếp đến điều, khoản, điểm cụ thể
- [ ] **Evaluation dashboard**: Precision@K, MRR, latency tracking theo thời gian

---

## Phase 4 — Multi-jurisdiction (Month 5–6)

**Focus**: Mở rộng ra Southeast Asia.

### Singapore
- Companies Act, MAS Payment Services Act, PDPA
- Fintech sandbox guidelines

### Thailand
- PDPA (Personal Data Protection Act 2019)
- Bank of Thailand crypto regulations
- Digital Economy Promotion Act

### Indonesia
- OJK (financial services) regulations
- GR 71/2019 (electronic systems)
- MoT e-commerce regulations

### Philippines
- BSP (central bank) virtual asset rules
- Data Privacy Act 2012
- Electronic Commerce Act

**Architecture change**: Each jurisdiction gets its own Qdrant collection. Query router detects jurisdiction from context, queries appropriate collection.

---

## Phase 5 — Platform (Month 7–12)

**Focus**: Themis như một platform, không chỉ là một tool.

### 5.1 Knowledge Base Marketplace
Teams chia sẻ curated knowledge bases với nhau:
- "Vietnam Fintech Compliance Pack" — 50 documents, pre-indexed
- "SEA Gaming Regulations Pack" — ToS của 20 game publishers
- "GDPR Compliance Pack" — EU data protection documents

### 5.2 API for Legal AI Applications
```
POST /v1/search          # Retrieval API
POST /v1/check           # Compliance check endpoint
POST /v1/compare         # Compare two jurisdictions
GET  /v1/updates         # New regulations since {date}
```

### 5.3 Webhook Notifications
```
When new regulation published → webhook → Slack/email notify
When regulation updated → re-index → notify affected teams
```

### 5.4 White-label
Law firms và legal tech companies deploy Themis với brand của họ, feed vào knowledge base riêng (client documents, internal memos).

---

## Phase 6 — Advanced AI (Year 2)

### 6.1 GraphRAG cho Legal
Legal documents có entity relationships phức tạp:
- Luật A sửa đổi Điều 5 của Luật B
- Nghị định X hướng dẫn thi hành Luật Y
- Thông tư Z áp dụng cho đối tượng defined trong Luật A và B

Knowledge graph capture những relationships này:
```
PCRT 2022 ──amends──► PCRT 2012 (Điều 3, 5, 8)
Decree 52 ──implements──► E-commerce Law
MiCA ──references──► FATF Recommendation 15
```

Query: "Quy định về KYC cho ví điện tử" → traverse graph → tổng hợp từ nhiều luật có liên quan.

### 6.2 Compliance Checker
```python
# Input: feature description
# Output: compliance gaps + relevant articles

result = themis.check_compliance(
    feature="Cho phép user rút tiền VND qua MoMo, tối đa 50M/ngày",
    jurisdictions=["vietnam"],
    domains=["fintech", "aml"]
)

# Returns:
# {
#   "compliant": false,
#   "gaps": [
#     {
#       "issue": "Yêu cầu báo cáo giao dịch > 300M VND/ngày",
#       "article": "Điều 22, Luật PCRT 2022",
#       "severity": "HIGH"
#     }
#   ]
# }
```

### 6.3 Regulation Monitoring
Auto-track Công báo Chính phủ, Cổng Thông tin pháp luật, EUR-Lex, FinCEN.gov — khi có văn bản mới liên quan đến domains đang track, tự động index và notify team.

---

## Long-term Vision (3 năm)

**Themis = Stripe cho legal compliance.**

Giống như:
- Developer không cần hiểu payment networks để accept credit card — dùng Stripe API
- Developer không cần đọc luật để biết feature của mình có compliant không — dùng Themis API

```python
# In year 3, this is what developers call:
result = await themis.is_compliant(
    feature_description="...",
    user_location="VN",
    transaction_amount=50_000_000,
    transaction_type="withdrawal"
)
# → { compliant: true/false, confidence: 0.92, citations: [...] }
```

---

## CV Progression cho Partner

| Phase | Có thể nói trong interview |
|---|---|
| Phase 1 | "Built hybrid RAG pipeline with LangChain, multilingual embeddings, achieving X% Precision@5 on Vietnamese legal text" |
| Phase 2 | "Implemented Google Drive connector with OAuth 2.0 and incremental sync, deployed cloud MCP endpoint" |
| Phase 3 | "Designed agentic RAG system with multi-step query decomposition and conflict detection" |
| Phase 4 | "Scaled system to 4 Southeast Asian jurisdictions with jurisdiction-routing and per-country Qdrant collections" |
| Phase 5 | "Architected legal AI platform serving multiple law firms via white-label deployment" |
| Phase 6 | "Implemented GraphRAG for legal entity relationships and automated compliance checking API" |
