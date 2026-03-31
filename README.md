# Themis ⚖️

> **AI-powered legal intelligence platform for Southeast Asian startups**
> Bring your entire legal knowledge base into Claude, Cursor, and every AI agent — via MCP.

---

## What is Themis?

Themis is an **MCP-native RAG server** that lets AI agents automatically retrieve legal knowledge when needed — without any manual prompting.

```
Google Drive ──┐
Local docs    ─┼──► Themis ──► Claude / Cursor / Copilot
Notion        ─┘   (MCP)      auto-cites legal sources on every answer
```

Built for startups in Vietnam and Southeast Asia navigating:
- Fintech / crypto regulations
- Gaming and digital goods compliance
- E-commerce and escrow laws
- Data privacy (Vietnam PDPD, GDPR)
- Tax obligations

---

## Why Themis?

| Problem | Current solutions | Themis |
|---|---|---|
| Legal docs scattered across Drive, Notion | Manual search | Auto-indexed, always searchable |
| AI agents hallucinate on legal questions | No grounding | Every answer cites source + article |
| RAG tools don't connect to AI agents natively | Chatbot UI | MCP — agent calls it automatically |
| Multilingual legal text (Vietnamese + English) | English-only tools | Multilingual-e5, supports both |

---

## Quick Start

```bash
git clone https://github.com/yourname/themis
cd themis

# Start full stack
docker compose up

# Index your documents
python pipeline/cli.py index ./knowledge

# Test search
python pipeline/cli.py search "quy định AML cho sàn crypto Việt Nam"

# Connect to Claude Code
# Add to ~/.claude/claude_desktop_config.json:
{
  "mcpServers": {
    "themis": {
      "command": "node",
      "args": ["/path/to/themis/mcp/server.js"],
      "env": { "THEMIS_API": "http://localhost:8000" }
    }
  }
}
```

---

## Architecture

```
┌─────────────────────────────────────────────┐
│              MCP Server            │
│   search · list_sources · get_document       │
└──────────────────┬──────────────────────────┘
                   │ HTTP
┌──────────────────▼──────────────────────────┐
│           AI Pipeline (Python)               │
│  LangChain · HuggingFace · Qdrant           │
│  Hybrid Search · Cross-encoder Reranking     │
└──────┬──────────────────────────────┬────────┘
       │                              │
┌──────▼──────┐              ┌───────▼────────┐
│   Qdrant    │              │  Google Drive  │
│ Vector DB   │              │  Notion / Local│
└─────────────┘              └────────────────┘
```

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| AI orchestration | **LangChain** | Industry standard, MultiQueryRetriever, ContextualCompression |
| Embeddings | **multilingual-e5-small** (HuggingFace) | Local, free, Vietnamese + English |
| Vector DB | **Qdrant** | Production-grade, hybrid search built-in |
| Reranking | **cross-encoder/ms-marco-MiniLM** | +15–30% precision, local |
| MCP Server | **Node.js + @modelcontextprotocol/sdk** | Native MCP support |
| Connectors | **Google Drive API, Notion API** | Where legal docs actually live |
| Infra | **Docker Compose** | One-command setup |

---

## MCP Tools

### `search`
Semantic + keyword hybrid search across entire knowledge base.

### `list_sources`
List all indexed documents with chunk counts.

### `get_document`
Retrieve full content of a specific document.

### `refresh`
Trigger re-sync of a connector (Google Drive, Notion).

---

## Supported Document Formats

`.pdf` · `.docx` · `.md` · `.html` · `.txt` · `.png` · `.jpg` (OCR) · `.json` · `.yaml`

---

## Knowledge Base Structure

```
knowledge/
  vietnam/        # Luật Việt Nam (PCRT, PDPD, thuế, TMĐT)
  international/  # FATF, GDPR, MiCA, FinCEN
  gaming-tos/     # Steam, Blizzard, Epic ToS
  escrow-regs/    # Quy định escrow, chuyển tiền
  tax/            # Thuế TNCN, TNDN, VAT
```

---

## License

MIT — free to self-host. Cloud version coming soon.
