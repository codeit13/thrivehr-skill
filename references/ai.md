# AI Services Standards

## Canonical Python AI Stack


| Component      | Standard                                                         | Do not use                          |
| -------------- | ---------------------------------------------------------------- | ----------------------------------- |
| HTTP framework | **FastAPI**                                                      | Flask, Django, bare ASGI            |
| Validation     | **Pydantic v2**                                                  | Pydantic v1, manual dict validation |
| ORM            | **SQLAlchemy + Alembic**                                         | Raw SQL in routers, Django ORM      |
| Lint           | **Ruff**                                                         | pylint-only                         |
| Logging        | **Loguru**                                                       |                                     |
| Testing        | **Pytest + httpx** `AsyncClient`                                 | unittest-only                       |
| API docs       | **FastAPI OpenAPI** (`/docs`, `/redoc`) — **password-protected** | Public docs                         |


---

## Architecture

**Core rule**: No AI service calls models directly. Everything goes through **ai-orchestrator**.

```
Business Services (Node.js/Fastify)
    │ HTTP / Message Queue
    ▼
AI Consumer Services (Python/FastAPI)
    │ HTTP (internal)
    ▼
ai-orchestrator (FastAPI + LangChain)
    │
    ├── AWS Bedrock (Claude, Llama 70B)
    ├── Self-hosted (Gemma, Llama, Qwen)
    └── Google APIs (Embeddings)
```

### Consumer Services — Do / Do Not

**Do**: receive request, prepare context, call orchestrator, validate output, persist, return response.

**Do NOT**: import LangChain, hold API keys, call model providers directly, manage prompts, track tokens.

### AI Services Inventory

Service names follow [core.md](core.md) (`ai-{servicename}`). The `source` field on orchestrator requests uses the same name (e.g. `ai-resume`).

| Service | Responsibility | Calls Orchestrator For |
|---------|---------------|------------------------|
| `ai-orchestrator` | Model routing, LangChain, token tracking, prompt mgmt | N/A — gateway only |
| `ai-resume` | Parse resumes, extract skills/experience, structure data | LLM completion, embeddings |
| `ai-skillmatch` | Match candidate skills vs job requirements, scoring | Embeddings, vector search, LLM scoring |
| `ai-interview` | Generate questions, evaluate responses, score answers | LLM completion |
| `ai-recommender` | Recommend jobs to candidates / candidates to recruiters | Vector search, LLM ranking |
| `ai-chatbot` | Onboarding assistant, FAQ, guided flows | LLM completion (conversational) |

### Consumer Request Flow

```
Request (from business service)
    │
    ▼
Router (FastAPI) — validate input only
    │
    ▼
Service Layer
    ├── Prepare input / context
    ├── Call OrchestratorClient (HTTP)     ← app/common/orchestrator_client.py
    ├── Validate LLM output (Pydantic)
    ├── Post-process (scoring, ranking, formatting)
    └── Persist results (Repository)
    │
    ▼
Response (envelope to business service)
```

Every AI consumer uses the shared `OrchestratorClient`. No direct model calls.

---

## Project Structure (FastAPI)

```
ai-{servicename}/
├── app/
│   ├── __init__.py
│   ├── main.py                            # FastAPI app, lifespan
│   ├── common/
│   │   ├── cache/
│   │   │   ├── cache_interface.py
│   │   │   └── in_memory_cache.py         # [CACHE_SWAP]
│   │   ├── config/env_config.py
│   │   ├── database/
│   │   │   ├── session.py
│   │   │   └── base.py
│   │   ├── middleware/
│   │   │   ├── correlation_id.py
│   │   │   └── rate_limit.py
│   │   ├── exceptions/handlers.py
│   │   ├── schemas/response.py            # Envelope schema
│   │   ├── orchestrator_client.py       # HTTP client to ai-orchestrator (consumers only)
│   │   └── utils/
│   ├── settings/
│   │   ├── router.py
│   │   ├── service.py
│   │   ├── repository.py
│   │   ├── models.py
│   │   └── schemas.py
│   ├── health/
│   │   └── router.py
│   ├── modules/{domain}/
│   │   ├── router.py
│   │   ├── service.py
│   │   ├── repository.py
│   │   ├── models.py
│   │   └── schemas.py
│   └── events/
│       ├── queue_producer.py
│       └── queue_consumer.py
├── migrations/
│   └── versions/
├── tests/
│   ├── unit/
│   └── integration/
├── alembic.ini
├── .env.example
├── pyproject.toml
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── README.md
```

---

## Orchestrator Design

### Endpoints

```
POST /ai-orchestrator/api/v1/llm/complete
POST /ai-orchestrator/api/v1/embeddings/generate
POST /ai-orchestrator/api/v1/vectors/search
POST /ai-orchestrator/api/v1/vectors/upsert
POST /ai-orchestrator/api/v1/prompts/{use_case}
GET  /ai-orchestrator/api/v1/prompts/{use_case}
GET  /ai-orchestrator/api/v1/usage/summary
GET  /ai-orchestrator/api/v1/health
POST /ai-orchestrator/api/v1/admin/settings/reload   # hot-reload model/prompt settings
```

### LLM Completion Request/Response

```json
// Request
{
  "prompt": "string",
  "system_prompt": "string (optional — overrides stored prompt)",
  "model_tier": "fast | balanced | powerful",
  "temperature": 0.3,
  "max_tokens": 2000,
  "response_format": "text | json",
  "use_case": "resume_parse | interview_eval | skill_match | recommend | chat",
  "source": "ai-resume",
  "correlation_id": "uuid",
  "context": []
}

// Response
{
  "success": true,
  "data": {
    "content": "...",
    "model_used": "claude-sonnet-4-20250514",
    "tokens": { "input": 450, "output": 320, "total": 770 },
    "latency_ms": 1200
  },
  "meta": { "timestamp": "ISO8601", "requestId": "uuid" }
}
```

- `context`: optional conversation history / RAG chunks (orchestrator formats into prompt)
- `system_prompt`: escape hatch — overrides stored prompt for that `use_case`

### Embeddings Request

```json
{
  "texts": ["string"],
  "model": "text-embedding-004",
  "source": "ai-resume",
  "correlation_id": "uuid"
}
```

### Model Tiers (configurable via Settings Table)


| Tier       | Models                         | Use for                               |
| ---------- | ------------------------------ | ------------------------------------- |
| `fast`     | Gemma 2B/4B, Qwen 3B           | Extraction, classification            |
| `balanced` | Llama 3.1 8B, Gemma 7B         | Structured output, moderate reasoning |
| `powerful` | Claude Sonnet/Haiku, Llama 70B | Complex reasoning, evaluation         |


### Model Routing Settings (Settings Table)

| Setting Key | Example Value | Purpose |
|-------------|---------------|---------|
| `model_tier_fast` | `gemma-4b` | Model for `fast` tier |
| `model_tier_balanced` | `llama-3.1-8b` | Model for `balanced` tier |
| `model_tier_powerful` | `claude-sonnet` | Model for `powerful` tier |
| `use_case_resume_parse_tier` | `balanced` | Default tier for resume parsing |
| `use_case_interview_eval_tier` | `powerful` | Default tier for interview evaluation |
| `use_case_skill_match_tier` | `fast` | Default tier for skill matching |
| `use_case_recommend_tier` | `balanced` | Default tier for recommendations |
| `use_case_chat_tier` | `fast` | Default tier for chatbot |
| `embedding_model` | `text-embedding-004` | Embedding model for all vectorization |
| `embedding_dimension` | `768` | Vector dimension size |

### Swap Example (no redeploy)

1. Update setting: `use_case_resume_parse_tier` → `powerful`
2. Call `POST /ai-orchestrator/api/v1/admin/settings/reload`
3. Done — no code change

---

## Orchestrator Client (shared by all AI consumers)

Location: `app/common/orchestrator_client.py` in every AI consumer service.

```python
# app/common/orchestrator_client.py
class OrchestratorClient:
    def __init__(self, base_url: str, service_name: str):
        self.base_url = base_url
        self.service_name = service_name

    async def complete(self, prompt: str, use_case: str, model_tier: str = "balanced", **kwargs) -> LLMResponse:
        ...

    async def generate_embeddings(self, texts: list[str]) -> list[list[float]]:
        ...

    async def vector_search(self, collection: str, query_embedding: list[float], top_k: int = 10) -> list[SearchResult]:
        ...
```

---

## Prompt Management

- Prompts versioned and stored — never hardcoded in consumer code
- Simple prompts: Settings Table (`value_type=JSON`)
- Complex prompts: file-based (`prompts/{use_case}/v{n}.txt`) in **ai-orchestrator** repo
- Consumer passes `use_case`; orchestrator picks the prompt
- Consumer may pass `system_prompt` to override stored prompt (escape hatch only)
- Orchestrator loads prompts on startup (same hot-reload pattern as settings)
- All prompt changes reviewable (file-based = PR, settings-based = audit log)

### Prompt Structure (stored in orchestrator)

```json
{
  "use_case": "resume_parse",
  "version": "1.2",
  "system_prompt": "You are a resume parser...",
  "user_prompt_template": "Extract skills from: {resume_text}",
  "output_format": "json",
  "output_schema": { "skills": ["string"], "experience_years": "number" }
}
```

---

## Model Access Control

- Only **ai-orchestrator** holds model API keys / cloud credentials
- AI consumer services authenticate to orchestrator via **internal service token** (env: infra only, never in Settings Table)
- Pass token on every orchestrator request (e.g. `Authorization: Bearer <internal_service_token>`)
- No model credentials in any consumer service

---

## LangChain (Orchestrator Only)


| Feature | Component                                  | Purpose                        |
| ------- | ------------------------------------------ | ------------------------------ |
| Prompts | `PromptTemplate`, `ChatPromptTemplate`     | Versioned prompts              |
| Parsing | `PydanticOutputParser`, `JsonOutputParser` | Structured output              |
| Models  | `ChatBedrock`, `ChatOllama`, `ChatOpenAI`  | Model abstraction              |
| Chains  | `LLMChain`, `SequentialChain`              | Multi-step reasoning           |
| Retry   | `OutputFixingParser`                       | Auto-retry on malformed output |


**Not used for**: Agents (unpredictable), autonomous tool calling (security), cross-session memory.

---

## Self-Hosted Qdrant (orchestrator-owned)

- Only **ai-orchestrator** performs upsert/search against Qdrant
- Connection and collection names: Settings Table — never hardcode in consumers
- AI consumer services use orchestrator vector endpoints — never embed Qdrant client
- Health/readiness includes Qdrant connectivity check
- No Qdrant Cloud / managed SaaS unless explicitly approved

### Collections (default names — override via settings)

| Collection | Used By | Content |
|------------|---------|---------|
| `TM_EXT_RESUMES` | `ai-resume`, `ai-skillmatch` | Resume embeddings |
| `TM_EXT_JOBS` | `ai-recommender`, `ai-skillmatch` | Job description embeddings |
| `TM_EXT_SKILLS` | `ai-skillmatch` | Skill taxonomy embeddings |

Map to settings keys: `qdrant_collection_resumes` → `TM_EXT_RESUMES`, etc. Qdrant collection names use the **`TM_EXT_` prefix** (ALL CAPS), consistent with database naming.

### Vector Operations (via Orchestrator)

```json
// Upsert — POST /ai-orchestrator/api/v1/vectors/upsert
{
  "collection": "TM_EXT_RESUMES",
  "vectors": [
    { "id": "candidate-uuid", "embedding": [0.1, 0.2], "metadata": { "skills": ["python"] } }
  ],
  "source": "ai-resume",
  "correlation_id": "uuid"
}

// Search — POST /ai-orchestrator/api/v1/vectors/search
{
  "collection": "TM_EXT_RESUMES",
  "query_embedding": [0.1, 0.2],
  "top_k": 10,
  "filters": { "skills": { "$in": ["python", "fastapi"] } },
  "source": "ai-skillmatch",
  "correlation_id": "uuid"
}
```

### Qdrant Settings

| Setting Key | Example Value | Purpose |
|-------------|---------------|---------|
| `qdrant_host` | `qdrant.internal:6333` | Qdrant connection |
| `qdrant_collection_resumes` | `TM_EXT_RESUMES` | Resume collection (swappable) |
| `qdrant_collection_jobs` | `TM_EXT_JOBS` | Jobs collection |
| `qdrant_collection_skills` | `TM_EXT_SKILLS` | Skills collection |
| `vector_similarity_threshold` | `0.75` | Minimum score for a match |
| `vector_top_k_default` | `10` | Default result count |

---

## PII Privacy Guard (`ai_privacy_guard`)

Reusable module in the orchestrator pipeline — runs before external API calls.


| Mode              | Tool                                  | When                          |
| ----------------- | ------------------------------------- | ----------------------------- |
| CPU / self-hosted | **Microsoft Presidio** (text + image) | Default for on-prem           |
| CPU lightweight   | **GLiNER / GLiNER PII Edge**          | Zero-shot, consumer hardware  |
| Cloud (AWS)       | **Bedrock Guardrails**                | When using Bedrock models     |
| Cloud (Google)    | **Google Sensitive Data Protection**  | When using Gemini/Google APIs |


Rules:

- Strip PII (Aadhaar, PAN, phone, email) BEFORE external LLM calls
- Self-hosted models may receive PII (data stays in infra)
- Setting: `strip_pii_for_external_models = true`
- Never log PII in any service

---

## Observability & Token Tracking

All model calls logged to **`TM_EXT_AI_ORCHESTRATOR_USAGE`** table AND traced via **Langfuse**.

### Per-Call Log Shape (every LLM / embedding call)

```json
{
  "timestamp": "ISO8601",
  "source": "ai-resume",
  "use_case": "resume_parse",
  "model_used": "llama-3.1-8b",
  "model_tier": "balanced",
  "tokens_input": 450,
  "tokens_output": 320,
  "tokens_total": 770,
  "latency_ms": 1200,
  "status": "success",
  "correlation_id": "uuid",
  "estimated_cost_usd": 0.0012
}
```

### Usage Table Fields (`TM_EXT_AI_ORCHESTRATOR_USAGE`)

| Field | Type | Purpose |
|-------|------|---------|
| `id` | UUID | Primary key |
| `source` | String | Calling service (e.g. `ai-resume`) |
| `use_case` | String | What it was used for |
| `model_used` | String | Actual model that served the request |
| `model_tier` | String | Tier requested (`fast` / `balanced` / `powerful`) |
| `tokens_input` | Integer | Input tokens |
| `tokens_output` | Integer | Output tokens |
| `tokens_total` | Integer | Total tokens |
| `latency_ms` | Integer | Response time |
| `status` | String | `success` / `error` / `timeout` |
| `estimated_cost_usd` | Decimal | Cost estimate from model pricing settings |
| `correlation_id` | String | Request trace (match `X-Correlation-Id`) |
| `created_at` | Timestamp | When the call happened (UTC) |

### Budget Settings (Settings Table)

| Setting Key | Example Value | Purpose |
|-------------|---------------|---------|
| `ai_resume_daily_token_limit` | `500000` | Max tokens/day for `ai-resume` |
| `ai_skillmatch_daily_token_limit` | `200000` | Max tokens/day for `ai-skillmatch` |
| `ai_interview_daily_token_limit` | `300000` | Max tokens/day for `ai-interview` |
| `ai_recommender_daily_token_limit` | `200000` | Max tokens/day for `ai-recommender` |
| `ai_chatbot_daily_token_limit` | `400000` | Max tokens/day for `ai-chatbot` |
| `budget_alert_threshold_percent` | `80` | Alert when usage hits this % |
| `budget_block_at_limit` | `true` | Block calls at 100% (else alert only) |
| `model_cost_per_1k_tokens_claude` | `0.003` | Cost estimation |
| `model_cost_per_1k_tokens_llama_8b` | `0.0002` | Cost estimation |

Legacy setting keys may use `TM_EXT_AI_*` prefix in existing deployments — prefer `ai_{servicename}_*` for new keys.

### Budget Alerting Behavior

- At **80%** of daily limit → log WARNING, alert admin
- At **100%** → if `budget_block_at_limit=true`, return error code **`TOKEN_BUDGET_EXCEEDED`** (do not call model)
- Daily reset at **midnight UTC**
- All thresholds configurable via Settings Table + reload

### Langfuse Integration

- Export traces: latency, cost, prompt/response content
- LangSmith remains optional dev-only
- Dashboard for prompt debugging and regression detection

---

## AI Evaluation Strategy

Every AI workflow MUST have evaluation beyond "it works in the UI":

- **Golden datasets**: curated input/output pairs per use case
- **Regression evals**: run on every prompt or model change
- **Schema pass rate**: percentage of outputs passing Pydantic validation
- **Edge-case suites**: adversarial inputs, boundary conditions
- **Checklist**: new AI feature → eval suite entry in `tests/`

---

## Prompt Injection Prevention

- System prompts hardened and orchestrator-managed (not user-editable)
- Prompt hierarchy: system > developer > user — system prompt always takes precedence
- User input sanitized before injection into prompts
- Never execute LLM output as code
- Optional: LLM-based input/output classifiers for detecting injection attempts

### Output Validation (required for `response_format=json`)

1. Parse LLM output and validate against Pydantic schema for the `use_case`
2. On validation failure → **retry once** with stricter prompt / `OutputFixingParser` (orchestrator)
3. If still invalid → return structured error; **never** pass unvalidated output to downstream services or DB

Consumer services MUST re-validate orchestrator JSON responses before persisting.

---

## Output Safety (Poison / Harm Detection)

- Output moderation layer: blocklist + classifier
- Topics: self-harm, abuse, discrimination, reputational harm
- Do not rely solely on the model to be "safe" — add explicit guardrails
- Log flagged outputs for review

---

## Per-User Abuse Controls

- AI token rate limits per user (Settings Table)
- Confidence scoring on outputs
- Anomaly detection for suspicious patterns (prompt injection, manipulated inputs)
- Block or flag users exceeding thresholds

---

## FastAPI Swagger (password-protected)

Protect `/docs`, `/redoc`, `/openapi.json` with HTTP Basic auth. See [assets/fastapi-swagger-auth-template.md](../assets/fastapi-swagger-auth-template.md).

Credentials: `TM_EXT_SWAGGER_USER` / `TM_EXT_SWAGGER_PASSWORD` — same convention as Node services.