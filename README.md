# ARC — Autonomous Research Companion · Backend

> A modular research intelligence backend. Not a chatbot. A structured research operating system.

---

## Architecture

```
POST /research/start
        │
        ▼
  ResearchOrchestrator
        │
        ├── [1] LiteratureDiscoveryService
        │         ├── Semantic Scholar API
        │         └── arXiv API
        │         └── Normalize + Deduplicate
        │
        ├── [2] ExtractionService (LLM)
        │         └── Per-paper: problem, method, dataset, findings, limitations
        │
        ├── [3] ClusteringService (Deterministic)
        │         └── Keyword → Theme mapping
        │
        ├── [4] ContradictionDetectionService (LLM + Deterministic)
        │         └── Cross-group finding comparison
        │
        ├── [5] GapDetectionService (Deterministic + LLM formalization)
        │         └── Signal collection → LLM statement generation
        │
        ├── [6] ConfidenceScoringService (Deterministic formula)
        │         └── Weighted: support count, citations, recency, agreement
        │
        └── [7] BlueprintService (LLM + deterministic fallback)
                  └── Structured research outline generation

All stages persist to: /data/session-{uuid}.json
Frontend polls: GET /research/:id
```

---

## Folder Structure

```
arc-server/
├── package.json
├── .env.example
└── server/
    ├── server.js                      ← Express entry point
    ├── data/                          ← Session JSON files (gitignored)
    ├── routes/
    │   └── research.routes.js         ← All API endpoints
    ├── services/
    │   ├── research.orchestrator.js   ← Pipeline coordinator
    │   ├── literature.service.js      ← Phase 3: Paper discovery
    │   ├── extraction.service.js      ← Phase 4: LLM extraction
    │   ├── clustering.service.js      ← Phase 5: Theme clustering
    │   ├── contradiction.service.js   ← Phase 5: Contradiction detection
    │   ├── gap.service.js             ← Phase 5: Gap detection
    │   ├── confidence.service.js      ← Phase 5: Confidence scoring
    │   └── blueprint.service.js       ← Phase 5: Blueprint generation
    └── utils/
        ├── fileStorage.js             ← Session CRUD (JSON files)
        └── llmClient.js               ← LLM wrapper with retry + validation
```

---

## Setup

```bash
# 1. Clone and install
cd arc-server
npm install

# 2. Configure environment
cp .env.example .env
# Fill in ANTHROPIC_API_KEY

# 3. Run dev server
npm run dev
# → http://localhost:4000
```

---

## API Reference

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/research/start` | Start research session + run pipeline |
| `GET` | `/research/:id` | Get full session data (poll for progress) |
| `GET` | `/research/:id/status` | Lightweight status check |
| `GET` | `/research` | List all sessions |
| `DELETE` | `/research/:id` | Delete a session |
| `POST` | `/research/:id/simulate-week` | Simulate weekly update |

### POST /research/start

```json
{ "topic": "Low-cost AI models for crop disease detection in small-scale farms" }
```

Returns:
```json
{ "sessionId": "uuid", "status": "initializing" }
```

### GET /research/:id

Poll this until `status === "complete"`. Returns full dashboard data:

```json
{
  "status": "complete",
  "stats": { "paperCount": 32, "gapCount": 5, ... },
  "papers": [...],
  "themes": [...],
  "gaps": [...],
  "contradictions": [...],
  "claims": [...],
  "blueprint": { ... }
}
```

---

## Session Status Flow

```
initializing → fetching → extracting → analyzing → finalizing → complete
                                                              ↘ error
```

---

## LLM Usage

LLM is only called for:
1. **Structured extraction** per paper abstract (strict JSON schema)
2. **Contradiction detection** across methodology groups
3. **Gap formalization** from collected signals
4. **Blueprint generation** (with deterministic fallback)

Every LLM response is:
- Validated against a schema before storing
- Retried up to 3× on failure
- Replaced with deterministic fallback if all retries fail

**No uncontrolled LLM output is ever stored.**

---

## Confidence Score Formula

```
Score = (supportCount × 0.40)
      + (citationWeight × 0.25)
      + (recencyWeight × 0.20)
      + (agreementRatio × 0.15)
```

All components are explainable and stored in `scoreBreakdown` per claim.

---

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | Server port | `4000` |
| `ANTHROPIC_API_KEY` | Anthropic API key | required |
| `LLM_MODEL` | Model to use | `claude-haiku-4-5` |
| `SEMANTIC_SCHOLAR_API_KEY` | S2 API key (optional, for higher rate limits) | — |
| `LITERATURE_MAX_PAPERS` | Max papers per session | `40` |
| `LLM_RETRY_LIMIT` | LLM retry attempts | `3` |
