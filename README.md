# AI-Research-Assistant : Multi-Agent Context Engineering Pipeline

## What It Does

Upload a research paper (or any PDF document), ask a question in plain language, and receive a grounded, cited answer drawn from four independent sources — simultaneously:

| Source | What It Searches |
|--------|-----------------|
| 📄 **Document (RAG)** | The PDF you uploaded, semantically chunked and embedded |
| 🧠 **Memory** | Your conversation history from earlier in the session |
| 🌐 **Web** | Live web search for recent information |
| 📚 **ArXiv** | Academic papers related to your query |

Before generating a response, an **evaluator agent** scores each source for relevance to your question and discards low-signal context. Only what matters reaches the final model. This is the core of the context engineering approach.

The final answer includes full citations: document page numbers, chunk indices, URLs, and confidence scores.

---

## Architecture

```
Your Question
     │
     ▼
┌─────────────────────────────────────────────────────┐
│              4 Agents Run in Parallel                │
│                                                     │
│  📄 RAG Agent        → your uploaded PDF            │
│  🧠 Memory Agent     → your chat history            │
│  🌐 Web Agent        → live Firecrawl search        │
│  📚 ArXiv Agent      → academic paper search        │
└─────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────┐
│  Evaluator Agent                     │
│  Scores each source 0–1              │
│  Filters out irrelevant context      │
└──────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────┐
│  Synthesizer Agent                   │
│  Combines filtered context           │
│  Generates final response            │
│  Attaches citations + confidence     │
└──────────────────────────────────────┘
     │
     ▼
Answer  +  Sources  +  Confidence Scores
```

The agent framework is **CrewAI Flows**, which handles parallel execution and state management across the pipeline.

---

## Technology Stack

| Layer | Technology | Role |
|-------|------------|------|
| **Agent Orchestration** | [CrewAI Flows](https://www.crewai.com/) | Multi-agent pipeline with parallel execution |
| **Document Parsing** | [TensorLake](https://tensorlake.ai/) | PDF → structured chunks with section-level metadata |
| **Embeddings** | [Voyage AI](https://www.voyageai.com/) Context 3 | Contextualized semantic vectors (1024-dim) |
| **Vector Database** | [Milvus Lite](https://milvus.io/) | Local similarity search over document chunks |
| **Conversation Memory** | [Zep Cloud](https://www.getzep.com/) | Persistent memory with graph-based representation |
| **Web Search** | [Firecrawl](https://www.firecrawl.dev/) | Real-time web content extraction |
| **Academic Search** | [ArXiv API](https://arxiv.org/) | Free, open academic paper discovery |
| **Response Generation** | OpenAI GPT-4o-mini | Structured JSON output with citations |
| **UI** | Streamlit | Web interface for uploads and chat |

---

## Key Design Decisions

### 1. Context Engineering Over Prompt Engineering
The evaluator agent filters retrieved content *before* it reaches the synthesizer. The language model never sees irrelevant noise — only scored, relevant context. This produces more accurate answers than naive RAG approaches that dump all retrieved chunks into one prompt.

### 2. Parallel Context Gathering
All four sources are queried simultaneously using CrewAI's parallel execution. Latency stays manageable as the number of sources grows.

### 3. Structured Outputs Throughout
Every agent returns a Pydantic-validated JSON structure. The evaluator produces a `ContextEvaluationResult`. The synthesizer produces a schema-enforced response. No hallucinated formatting, no brittle string parsing.

### 4. Full Citation Traceability
Every answer includes citations down to the document page number and chunk index. You can see exactly which sentence in which section the answer came from.

### 5. Graceful Degradation
When context is insufficient, the system returns `INSUFFICIENT_CONTEXT` rather than hallucinating. No confident-sounding wrong answers.

---

## Pipeline Flow

```
1.  User uploads PDF
          ↓
2.  TensorLake parses PDF → structured chunks
          ↓
3.  Voyage AI embeds each chunk → 1024-dim vectors
          ↓
4.  Vectors inserted into Milvus Lite (local DB)

—— Document processing complete ——

5.  User asks a question
          ↓
6.  Query saved to Zep Cloud memory (thread-scoped)
          ↓
7.  4 agents run in parallel:
      - RAGTool: embeds query → Milvus similarity search → top-k chunks
      - MemoryTool: retrieves Zep conversation summary
      - FirecrawlTool: web search → title + snippet + URL
      - ArXivTool: HTTP API → paper metadata
          ↓
8.  Evaluator agent scores each source 0–1 for relevance
    Low-relevance sources are dropped
          ↓
9.  Synthesizer agent receives only relevant context
    GPT-4o-mini generates structured JSON response
          ↓
10. Response + citations saved to Zep memory
          ↓
11. UI renders answer with source attribution cards
```

---

## Project Structure

```
context-engineering/
├── app.py                        # Streamlit web interface
├── pyproject.toml                # Python dependencies
│
├── config/
│   ├── agents/
│   │   └── research_agents.yaml  # Agent roles, goals, backstories
│   └── tasks/
│       └── research_tasks.yaml   # Task descriptions and expected outputs
│
├── src/
│   ├── workflows/
│   │   ├── flow.py               # ResearchAssistantFlow (main orchestrator)
│   │   ├── agents.py             # Agent factory from YAML config
│   │   └── tasks.py              # Task factory from YAML config
│   │
│   ├── rag/
│   │   ├── rag_pipeline.py       # Unified RAG: upload → embed → retrieve
│   │   ├── embeddings.py         # Voyage AI contextualized embeddings
│   │   └── retriever.py          # Milvus Lite vector database
│   │
│   ├── tools/
│   │   ├── rag_tool.py           # CrewAI tool wrapping RAG pipeline
│   │   ├── memory_tool.py        # CrewAI tool wrapping Zep memory
│   │   ├── web_search_tool.py    # Firecrawl web search tool
│   │   └── arxiv_tool.py         # ArXiv academic search tool
│   │
│   ├── document_processing/
│   │   └── doc_parser.py         # TensorLake PDF parser
│   │
│   ├── memory/
│   │   └── memory.py             # Zep Cloud conversation memory layer
│   │
│   └── generation/
│       └── generation.py         # OpenAI structured response generation
│
└── data/                         # Place your PDF documents here
```

---

## What It Does Not Do

- **No multi-format support** — PDFs only (no Word, PowerPoint, images)
- **No multi-user sessions** — single user per Streamlit instance
- **No production auth** — not designed as a multi-tenant system
- **No real-time ingestion** — documents are processed on upload, not continuously monitored
- **No fine-tuning** — uses off-the-shelf models throughout

This is a single-user, session-based assistant built for prototyping, research workflows, and demonstrating context engineering patterns.
