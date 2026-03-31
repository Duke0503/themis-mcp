# Themis — Product Requirements Document

**Version**: 1.0
**Status**: Active Development
**Target users**: Vietnamese startup developers, legal teams, compliance officers

---

## 1. Problem Statement

### 1.1 Context

Startups in Vietnam — especially in fintech, gaming, and e-commerce — operate in a complex, rapidly-evolving regulatory environment:

- **Vietnam**: PCRT 2022, PDPD (data protection), Circular 09 (crypto), Decree 52 (e-commerce), multiple tax circulars
- **International**: FATF recommendations, GDPR, MiCA, FinCEN guidance
- **Platform-specific**: Game publisher ToS (Steam, Blizzard, Epic) for marketplace operators

Legal consultation is expensive ($100–500/hour). Most teams cannot afford a full-time legal counsel. Developers and product managers end up making compliance decisions without proper context — creating significant business and legal risk.

### 1.2 Current Workaround (and Why It Fails)

Teams currently:
1. Google the regulation → land on outdated blog posts, not primary sources
2. Ask ChatGPT → hallucinated answers with no citations, no source verification
3. Hire a consultant for every question → expensive, slow, breaks developer flow

**Core failure**: AI models hallucinate on legal questions because they have no access to the actual regulatory documents.

### 1.3 Root Cause

No tool today provides:
- Grounded answers from **primary legal sources** (not summaries)
- **Source citations** at article level (not just document level)
- **Native AI agent integration** (via MCP — not chatbot UI)
- **Vietnamese law** coverage alongside international regulations

---

## 2. Target Users

### Primary: Startup CTO / Lead Developer
- Building a product that handles money, user data, or digital goods
- Needs to make quick compliance decisions during development
- Uses Claude Code or Cursor daily
- Does not have time to read 200-page legal documents

**Job to be done**: "When I'm building a feature that touches payments or user data, I need to know if it's legal in Vietnam before I ship — without stopping to hire a lawyer."

### Secondary: Compliance Officer (non-technical)
- Responsible for ensuring the company meets regulatory requirements
- Maintains the legal document library
- Needs to answer questions from product and engineering teams quickly

**Job to be done**: "I need to quickly find the specific article that governs our situation and share it with the dev team."

### Tertiary: Legal Consultant / Law Firm
- Building AI-assisted legal research tools for their clients
- Needs a self-hostable, customizable RAG backend

---

## 3. Goals

### MVP Goal (Month 1)
> **A developer using Claude Code can ask a compliance question and receive a cited, grounded answer from Vietnamese and international law — in under 3 seconds.**

### 6-Month Goal
> **Teams can connect their Google Drive legal library to Themis and query it from any AI agent, with permission-aware access control.**

### 12-Month Goal
> **Themis becomes the standard legal knowledge layer for AI agents in the Vietnamese startup ecosystem.**

---

## 4. User Stories

### US-001 — Developer queries during coding
```
AS a developer using Claude Code
I WANT to ask "Điều kiện KYC cho ví điện tử theo luật Việt Nam là gì?"
SO THAT I receive the exact legal requirement with article citation
WITHOUT leaving my editor
```
**Acceptance**: Response includes article number, law name, effective date, confidence score.

### US-002 — Compliance check before shipping
```
AS a product manager
I WANT to ask "Does our user data retention policy comply with Vietnam PDPD?"
SO THAT I can identify gaps before launch
WITHOUT reading the entire 40-page decree
```
**Acceptance**: Returns specific articles of PDPD relevant to data retention, with similarity scores.

### US-003 — Index team's Google Drive
```
AS a compliance officer
I WANT to connect our Google Drive legal folder to Themis
SO THAT the entire team can query our internal legal docs via AI
WITHOUT manually uploading files
```
**Acceptance**: OAuth connect → auto-sync → searchable in < 5 minutes.

### US-004 — Multilingual query
```
AS a Vietnamese developer
I WANT to ask in Vietnamese and receive answers citing both Vietnamese and English sources
SO THAT I understand both the local law and international standards
```
**Acceptance**: Vietnamese query returns results from both `vietnam/` and `international/` categories.

### US-005 — Reindex on document update
```
AS a compliance officer who updated a legal document in Google Drive
I WANT Themis to automatically detect the change and re-index
SO THAT the AI agent always searches current versions
```
**Acceptance**: File update detected within 60 seconds, re-indexed automatically.

---

## 5. Features (Prioritized)

### P0 — Must have for MVP

| ID | Feature | Description |
|---|---|---|
| F-001 | Hybrid semantic search | BM25 + dense vector + RRF fusion |
| F-002 | Source citations | Every result includes article, document, jurisdiction |
| F-003 | Confidence scoring | HIGH / MEDIUM / LOW per result |
| F-004 | MCP server | stdio transport, 4 tools |
| F-005 | PDF/DOCX/Markdown parsing | Core document formats |
| F-006 | Legal article chunking | Vietnamese Điều X + English Article X |
| F-007 | CLI indexing | index, search, status commands |
| F-008 | Docker self-host | One-command setup |
| F-009 | Local embeddings | No API key required |

### P1 — Month 2

| ID | Feature | Description |
|---|---|---|
| F-010 | Google Drive connector | OAuth 2.0, auto-sync |
| F-011 | Cross-encoder reranking | Second-pass precision improvement |
| F-012 | Notion connector | Pages and databases |
| F-013 | Incremental indexing | Only re-process changed files |
| F-014 | Multi-language detection | Route queries to best index |

### P2 — Month 3+

| ID | Feature | Description |
|---|---|---|
| F-015 | Permission propagation | Google Drive share settings respected |
| F-016 | Web dashboard | Index status, query logs, latency metrics |
| F-017 | MultiQueryRetriever | Generate 3 query variants, merge results |
| F-018 | Cloud MCP endpoint | SSE transport, remote access |
| F-019 | Evaluation dashboard | Precision@K, MRR tracking over time |

---

## 6. Non-Goals (MVP)

- No chatbot UI — MCP only
- No user authentication — single-tenant
- No real-time crawling — manual index trigger
- No billing / SaaS — open source self-hosted first
- No legal advice — information retrieval only, always disclaim

---

## 7. Success Metrics

| Metric | Target | How measured |
|---|---|---|
| Precision@5 (Vietnamese legal queries) | ≥ 70% | Manual evaluation, 50 test queries |
| Precision@5 (English legal queries) | ≥ 75% | Manual evaluation, 50 test queries |
| Search latency (p95, after warmup) | < 2s | CLI timing |
| Cold start time | < 5s | Docker startup timing |
| Setup time (clone → first search) | < 10 min | User test |

---

## 8. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Low retrieval quality for Vietnamese legal text | Medium | High | Multilingual-e5 + Vietnamese-specific chunking patterns |
| Google Drive API quota exhaustion | Low | Medium | Aggressive caching, batch sync |
| Legal documents become outdated | High | High | Document metadata includes effective_date, flag old docs |
| MCP spec breaking change | Low | High | Abstract transport layer, pin SDK version |
