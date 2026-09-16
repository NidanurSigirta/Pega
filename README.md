✈️ PEGA — RAG-Based AI Customer Support Agent for Pegasus Airlines
<p align="center"><i></i></p>

**CBOT Internship Case Study #1 — A production-style Retrieval-Augmented Generation (RAG) support agent built on n8n, orchestrating a full knowledge pipeline from ingestion to real-time customer conversation.**

> **TL;DR:** RAG-based Pegasus Airlines support agent — 3 cooperating n8n workflows, a 292-chunk vector store, Redis-backed session memory, 2 live API integrations (weather & currency), an 11-rule guardrail system prompt, and 25+ manually written test scenarios used to find and fix real failure modes (embedding dilution, negation reversal, hallucination).

---

## About the Project

PEGA is a RAG-based AI customer support agent developed as the first case study of the CBOT internship program, designed to answer Pegasus Airlines customer questions with grounded, up-to-date information rather than relying on a language model's raw knowledge.

The project was also the entry point into **RAG architecture and agent guardrail engineering**, built from the ground up: a raw knowledge base was micro-chunked, embedded, and indexed into a searchable vector store, then wired into a conversational agent capable of retrieving the most relevant context for every user query in real time — while separately maintaining Redis-backed session memory so the agent remembers each user's conversation history.

Beyond fulfilling the base requirements, this project was treated as an open engineering problem rather than a fixed checklist: several real failure modes (semantic search losing rare entities in large text blocks, the model inverting negative facts, occasional hallucinated recommendations) were discovered through systematic testing, root-caused, and fixed with targeted architectural changes — documented in detail below.

Rather than a single monolithic workflow, PEGA is architected as **three cooperating n8n workflows**, each with a distinct responsibility in the RAG lifecycle — separating knowledge ingestion, scheduled maintenance, and live conversation handling.

---

## Key Features

- **Retrieval-Augmented Generation:** Combines OpenAI embeddings with a Simple Vector Store so responses are grounded in real Pegasus Airlines knowledge, not model hallucination.
- **Micro-Chunked Knowledge Base:** 292 finely segmented knowledge items, restructured from an initial coarse layout after diagnosing a real retrieval failure (see *Embedding Dilution* below) — a concrete example of iterating on architecture based on evidence, not just intuition.
- **Live External Data:** Two real-time API integrations (weather and currency conversion) so the agent can answer time-sensitive questions instead of relying on static or outdated knowledge.
- **Session-Based Conversation Memory:** Redis-backed chat memory keeps track of each user's conversation history independently of the knowledge base, enabling multi-turn, context-aware dialogue.
- **Modular Workflow Architecture:** Ingestion, refresh, and conversation are fully decoupled into separate n8n workflows for maintainability and independent scheduling.
- **Automated Knowledge Refresh:** A scheduled scraping pipeline pulls live updates directly from the source website and re-syncs the vector store without manual re-ingestion.
- **Guardrail-Engineered Responses:** An 11-rule system prompt, refined through systematic testing, prevents hallucination, negation reversal, off-network recommendations, and off-topic answers — detailed in *Problems Found & Fixed*.
- **Containerized & Reproducible:** Redis and n8n run via Docker on a shared network, ensuring a consistent environment across machines.
- **Cost-Efficient Inference:** Powered by gpt-4o-mini at a low temperature (0.1) for consistent, low-latency, low-cost responses suited to a customer-support use case.

---

##  Workflow Architecture

### 1. Load Knowledge Base
Ingests the raw Pegasus Airlines knowledge base, splits it into 292 micro-chunks, generates OpenAI embeddings for each chunk, and writes them into the Simple Vector Store — establishing the retrieval foundation for the agent.

### 2. Auto Data Refresh
A scheduled workflow that scrapes the live source website (HTTP request → HTML extraction via CSS selector → content parsing) and appends newly discovered information to the vector store, keeping retrieved answers current as underlying information changes — without needing to rerun ingestion manually or wait for a manual data update.

### 3. Support Agent
The customer-facing conversational workflow. Receives a user query, retrieves the most relevant chunks from the vector store, consults Redis for prior conversation context, optionally calls live weather/currency tools, and generates a grounded response via gpt-4o-mini under an 11-rule guardrail system prompt — the core RAG loop in action.

---

## Engineering Deep Dive

This section documents problems that were not explicitly requested by the case brief, but were discovered through systematic testing and treated as engineering responsibilities of the role.

### Embedding Dilution → Micro-Chunking
Early testing revealed that queries about less-frequently-mentioned destinations (e.g. Russia, the Netherlands) were failing to retrieve any result, even though the information existed in the knowledge base. Root-causing this showed that large, continent-level text blocks were diluting the semantic weight of individual country names during embedding — a failure mode not obvious from the data itself, only from adversarial testing. The fix was to split destination data into four independent regional chunks (domestic / Europe / Asia / Africa) instead of one large block per continent, after which retrieval accuracy for rare entities reached 100% in testing.

### Problems Found & Fixed (via 25+ manual test scenarios)

| Problem | Root Cause | Fix |
|---|---|---|
| Hallucinated details (wrong baggage kg, check-in timing) | Model filled knowledge gaps from general training data | Rule enforcing tool-only sourcing + temperature lowered to 0.1 |
| Negation reversal (e.g. "cannot transfer" answered as "can transfer") | Model inverted negative facts across differently-phrased questions | Explicit SORU/CEVAP (Q/A) response format + dedicated negation-handling rule |
| Off-network destination recommendations | No boundary defined for the agent's knowledge scope | Explicit destination allowlist + guardrail rule |
| Irrelevant information appended to answers | RAG returned multiple chunks and the agent used all of them | Rule restricting the agent to only the topic actually asked |
| Incomplete check-in information | Source data only covered the online check-in path | Domestic/international check-in steps added with exact cutoff times |
| Conflicting scraped vs. curated data | An older manually-entered note contradicted newly scraped ground truth | Resolved by giving scraped (live) data precedence over static notes |

### Live Data Integration
Two HTTP Request tools were added to the agent: a weather API (OpenWeatherMap) and a currency conversion API (open.er-api.com, no authentication required). The system prompt explicitly forbids the agent from estimating either value — both must come from a live tool call, verified in testing by comparing agent responses against the raw API output.

### Validation
The final system was validated against 25+ manually written test scenarios spanning destination lookups, menu/price arithmetic, package eligibility rules, baggage limits, Fast Track exceptions, cultural notes, and both live-data tools — including a scraped-data-only scenario (Azerbaijan visa requirements) that was not present in the manually curated dataset, confirming the auto-refresh pipeline works end-to-end.

---

## Technologies Used

| Technology | Purpose |
|---|---|
| **n8n** | Workflow orchestration for ingestion, refresh, and conversation logic |
| **Docker** | Containerized, reproducible environment for Redis and n8n on a shared network |
| **Redis** | Session-based conversation memory for the support agent (independent of the vector store) |
| **Simple Vector Store** | Native n8n vector index for semantic chunk retrieval |
| **OpenAI Embeddings** | Vectorization of knowledge base chunks for semantic search |
| **gpt-4o-mini** | Response generation for the conversational agent, temperature 0.1 |
| **OpenWeatherMap API** | Live weather data for destination queries |
| **open.er-api.com** | Live currency exchange rate data |

---

## 📂 Project Structure

```
pega/
├── workflows/
│   ├── Pegasus - Load Knowledge Base.json      # KB ingestion & embedding workflow
│   ├── Pegasus - Auto Data Refresh.json         # Scheduled scraping & vector store sync
│   └── Pegasus - Support Agent.json             # Live customer conversation workflow
├── assets/
│   ├── Load Knowledge Base.png                  # Workflow screenshot
│   ├── Auto Data Refresh.png                    # Workflow screenshot
│   └── Support Agent.png                        # Workflow screenshot
└── README.md                                    # Project documentation
```

---

## Installation & Setup

### Prerequisites
- [n8n](https://n8n.io/) (self-hosted or desktop)
- Docker Desktop
- An OpenAI API key
- An OpenWeatherMap API key (free tier)

### 1. Start Redis via Docker
```bash
docker-compose up -d
```

### 2. Import the Workflows
In your n8n instance, import the three workflow files in this order:
1. `Pegasus - Load Knowledge Base.json`
2. `Pegasus - Auto Data Refresh.json`
3. `Pegasus - Support Agent.json`

### 3. Configure Credentials
Set your OpenAI API credentials, Redis connection details, and OpenWeatherMap API key inside each workflow's respective credential nodes.

### 4. Run the Ingestion Workflow
Execute **Load Knowledge Base** once to populate the vector store with the initial 292 chunks.

### 5. Activate the Agent
Activate **Support Agent** to start handling live queries, and **Auto Data Refresh** to keep the knowledge base current on schedule.

---

## Developer
**Nidanur Sıgırta** 

## 🛡️ License
© 2026 PEGA. All rights reserved.

*Grounding Conversations in Real Knowledge*
