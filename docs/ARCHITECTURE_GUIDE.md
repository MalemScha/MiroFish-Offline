# MiroFish-Offline — Complete Architecture & Usage Guide

## 1. Project Overview

**MiroFish-Offline** is a **multi-agent swarm intelligence simulation engine**. Upload a document (press release, financial report, policy draft, news article), and it generates hundreds of AI agents with unique personalities that simulate public reaction on social media — posts, arguments, opinion shifts — tracked round by round.

This is a fully local fork of [MiroFish](https://github.com/666ghj/MiroFish), replacing all cloud dependencies:

| Component | Original (Cloud) | This Fork (Local) |
|-----------|------------------|-------------------|
| **LLM** | DashScope / OpenAI | **Ollama** (qwen2.5, llama3, etc.) |
| **Graph DB** | Zep Cloud | **Neo4j CE 5.15** |
| **Embeddings** | Zep Cloud | **nomic-embed-text** via Ollama |
| **UI** | Chinese | **English** (1,000+ strings) |

---

## 2. System Architecture

### 2.1 High-Level Stack

```
┌──────────────────────────────────────────────────────────┐
│                     USER (Browser)                        │
│                 http://localhost:3000                      │
└────────────────────────┬─────────────────────────────────┘
                         │  HTTP (/api/* proxied → :5001)
┌────────────────────────▼─────────────────────────────────┐
│               FRONTEND — Vue 3 + Vite 7                   │
│   Port 3000 │ Axios │ D3.js graph viz │ Vue Router 4      │
└────────────────────────┬─────────────────────────────────┘
                         │  REST API
┌────────────────────────▼─────────────────────────────────┐
│              BACKEND — Flask (Python 3.11)                 │
│   Port 5001 │ 3 Blueprints │ 13 Services │ CORS enabled   │
└──────┬─────────────────────────────┬─────────────────────┘
       │                             │
┌──────▼──────────┐        ┌────────▼────────────┐
│  Neo4j CE 5.15   │        │     Ollama LLM       │
│  Port 7687/7474  │        │    Port 11434        │
│  Knowledge Graph │        │  qwen2.5:32b         │
│  + APOC plugin   │        │  nomic-embed-text    │
└─────────────────┘        └────────────────────┘
```

### 2.2 Backend Layers

```
Flask API Layer (3 blueprints)
  ├── /api/graph        → graph.py (20KB)
  ├── /api/simulation   → simulation.py (100KB)
  └── /api/report       → report.py (19KB)
         │
Service Layer (13 modules)
  ├── graph_builder.py           — Orchestrates knowledge graph creation
  ├── entity_reader.py           — Reads entities from graph for display
  ├── ontology_generator.py      — LLM generates entity/relation type schema
  ├── oasis_profile_generator.py — Generates agent profiles (47KB)
  ├── simulation_config_generator.py — Creates simulation parameters (40KB)
  ├── simulation_runner.py       — Runs OASIS simulation subprocess (72KB)
  ├── simulation_manager.py      — Tracks simulation state
  ├── simulation_ipc.py          — Inter-process communication
  ├── graph_memory_updater.py    — Writes simulation results to graph
  ├── graph_tools.py             — Hybrid search + graph query tools (57KB)
  ├── report_agent.py            — LLM-powered analysis agent (104KB)
  └── text_processor.py          — Text chunking utility
         │
Storage Layer (abstract interface)
  ├── graph_storage.py     — Abstract GraphStorage ABC
  ├── neo4j_storage.py     — Concrete Neo4j implementation (25KB)
  ├── neo4j_schema.py      — Cypher schema setup
  ├── embedding_service.py — Ollama nomic-embed-text (768d vectors)
  ├── ner_extractor.py     — LLM-based entity/relation extraction
  └── search_service.py    — Hybrid search (0.7 vector + 0.3 BM25)
         │
Utilities
  ├── llm_client.py  — OpenAI-compatible LLM wrapper
  ├── file_parser.py — PDF/MD/TXT parser
  ├── retry.py       — Retry logic for LLM calls
  └── logger.py      — Structured logging
```

### 2.3 Frontend Structure

```
frontend/src/
├── main.js                     # App bootstrap
├── App.vue                     # Root layout
├── router/index.js             # Route definitions
├── api/                        # Axios API modules
│   ├── index.js                #   Base config
│   ├── graph.js                #   /api/graph calls
│   ├── simulation.js           #   /api/simulation calls
│   └── report.js               #   /api/report calls
├── views/                      # Pages
│   ├── Home.vue + Home.css     #   Landing page
│   ├── Process.vue             #   5-step workflow container (54KB)
│   ├── MainView.vue            #   Dashboard
│   ├── SimulationView.vue      #   Sim config
│   ├── SimulationRunView.vue   #   Live sim progress
│   ├── ReportView.vue          #   Report display
│   └── InteractionView.vue     #   Agent chat
└── components/                 # Workflow steps
    ├── Step1GraphBuild.vue      #   Upload + graph build (18KB)
    ├── Step2EnvSetup.vue        #   Agent generation (72KB)
    ├── Step3Simulation.vue      #   Simulation control (40KB)
    ├── Step4Report.vue          #   Report gen + display (150KB)
    ├── Step5Interaction.vue     #   Agent chat UI (66KB)
    ├── GraphPanel.vue           #   D3 force-directed graph (41KB)
    └── HistoryDatabase.vue      #   Past sim browser (36KB)
```

---

## 3. The 5-Step Pipeline (In Depth)

### Step 1: Graph Build

**Purpose**: Extract structured knowledge from unstructured text.

1. **Upload** — User submits a document (PDF, MD, TXT; max 50MB)
2. **Parsing** — `file_parser.py` extracts raw text
3. **Chunking** — Text split into 500-char chunks with 50-char overlap
4. **Ontology Generation** — LLM reads the text and decides what entity types (Person, Organization, Event, Location…) and relation types (WORKS_FOR, SUPPORTS, OPPOSES…) are relevant
5. **NER/RE Extraction** — For each chunk, the LLM extracts entities and relationships using the ontology
6. **Graph Population** — Entities become nodes, relationships become edges in Neo4j
7. **Embedding** — Each entity/edge gets a 768-dimensional vector embedding for semantic search

**Result**: A knowledge graph representing all the key players, organizations, events, and their interconnections from your document.

### Step 2: Environment Setup

**Purpose**: Create the simulated world and its inhabitants.

1. **Entity Analysis** — System reads the knowledge graph
2. **Profile Generation** — LLM creates 100-500+ agent profiles, each with:
   - **Name & bio** (fictional but contextually appropriate)
   - **Personality** (Big Five traits, optimism/pessimism, aggression)
   - **Opinion bias** on the topic (-1.0 to +1.0)
   - **Reaction speed** (immediate reactor vs. slow thinker)
   - **Influence level** (how much their posts affect others)
   - **Memory** (knowledge of the document's events)
3. **Platform Config** — Choose Twitter-like or Reddit-like social platform
4. **Action Space** — Define what agents can do each round

### Step 3: Simulation

**Purpose**: Run the multi-agent social media simulation.

1. **OASIS Engine** — Powered by CAMEL-AI's OASIS framework
2. **Subprocess Execution** — Simulation runs in an isolated process (won't crash Flask)
3. **Round-by-Round**: Each round, every agent:
   - Observes the current social feed
   - Decides action based on personality + memory + current opinion
   - Posts, replies, likes, reposts, or does nothing
4. **Opinion Dynamics** — Agents influence each other; opinions shift over time
5. **Graph Update** — Results written back to Neo4j (new relationships, opinion trajectories)
6. **Real-time Status** — Frontend polls for progress

### Step 4: Report Generation

**Purpose**: Analyze what happened in the simulation.

1. **ReportAgent** — An LLM-powered analysis agent that:
   - Reviews all posts, replies, and sentiment trajectories
   - Selects a focus group of representative agents
   - "Interviews" them (asks why they posted what they posted)
   - Searches the knowledge graph for evidence (hybrid search: 0.7 vector + 0.3 BM25)
   - Generates a structured report
2. **Report Contents**:
   - Overall sentiment trend (positive/negative/neutral over time)
   - Key influencers and their impact
   - Topic propagation analysis
   - Opinion shift patterns
   - Focus group interview summaries

### Step 5: Interaction

**Purpose**: Post-simulation Q&A with individual agents.

- Select any agent from the simulation
- Chat with them — they retain full memory and personality
- Ask: "Why did you post that?", "What changed your mind?", "Who influenced you?"
- Powered by the same Ollama LLM with agent context injection

---

## 4. Key Design Patterns

| Pattern | Implementation | Why |
|---------|---------------|-----|
| **Abstract Storage** | `GraphStorage` ABC → `Neo4jStorage` | Swap graph DB by implementing one class |
| **Dependency Injection** | `app.extensions['neo4j_storage']` | No global singletons, testable |
| **Subprocess Isolation** | `simulation_ipc.py` | Crashed simulation ≠ crashed server |
| **Hybrid Search** | 0.7 vector + 0.3 BM25 | Better recall than vector-only |
| **OpenAI-Compatible API** | All LLM calls via `/v1/chat/completions` | Swap Ollama for any provider |
| **Blueprint Architecture** | 3 Flask blueprints | Clean API separation |

---

## 5. Configuration Reference

All settings via `.env` (copy from `.env.example`):

| Variable | Default | Purpose |
|----------|---------|---------|
| `LLM_API_KEY` | `ollama` | Any non-empty string for Ollama |
| `LLM_BASE_URL` | `http://localhost:11434/v1` | OpenAI-compatible endpoint |
| `LLM_MODEL_NAME` | `qwen2.5:32b` | LLM model name |
| `NEO4J_URI` | `bolt://localhost:7687` | Neo4j connection |
| `NEO4J_USER` | `neo4j` | Neo4j username |
| `NEO4J_PASSWORD` | `mirofish` | Neo4j password |
| `EMBEDDING_MODEL` | `nomic-embed-text` | Embedding model |
| `EMBEDDING_BASE_URL` | `http://localhost:11434` | Ollama embedding endpoint |
| `OASIS_DEFAULT_MAX_ROUNDS` | `10` | Simulation rounds |
| `REPORT_AGENT_MAX_TOOL_CALLS` | `5` | Max report agent tool calls |
| `REPORT_AGENT_TEMPERATURE` | `0.5` | Report creativity level |

---

## 6. Docker Deployment

### Services

| Service | Image | Ports | Purpose |
|---------|-------|-------|---------|
| `mirofish` | Built from `Dockerfile` | 3000, 5001 | Flask + Vite app |
| `neo4j` | `neo4j:5.15-community` | 7474, 7687 | Knowledge graph |
| `ollama` | `ollama/ollama:latest` | 11434 | LLM + embeddings |

### Quick Start

```bash
git clone https://github.com/nikmcfly/MiroFish-Offline.git
cd MiroFish-Offline
cp .env.example .env
docker compose up -d
docker exec mirofish-ollama ollama pull qwen2.5:32b
docker exec mirofish-ollama ollama pull nomic-embed-text
# Open http://localhost:3000
```

---

## 7. Hardware Requirements

| Tier | RAM | GPU VRAM | Model | Performance |
|------|-----|----------|-------|-------------|
| Minimal | 8 GB | CPU only | qwen2.5:3b | Slow, basic quality |
| Light | 16 GB | 6-8 GB | qwen2.5:7b | Small graphs |
| Standard | 32 GB | 12-16 GB | qwen2.5:14b | Most use cases |
| Power | 64 GB | 24+ GB | qwen2.5:32b | Full quality, fast |

---

## 8. API Endpoints Summary

### `/api/graph`
- Document upload + parsing
- Knowledge graph creation
- Entity/relationship CRUD
- Graph visualization data

### `/api/simulation`
- Agent profile generation
- Simulation config creation
- Simulation execution + status
- Agent browsing

### `/api/report`
- Report generation trigger
- Report status polling
- Report retrieval
- Focus group interview results

### `/health`
- Simple health check (`GET /health` → `{"status": "ok"}`)
