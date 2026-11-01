# aria × Creando Tu Matrix — Agent Offers Wiring Map

> **EN** — How this RAG stack backs the three agent offers of
> [Creando Tu Matrix](https://creandotumatrix.com), the "Day 2" AI operations
> firm (ES/EN, LATAM + US). Each track gets: what aria provides, what gets
> delivered to the client, and the concrete recipe.
>
> **ES** — Cómo este stack RAG respalda las tres líneas de ofertas con agentes
> de [Creando Tu Matrix](https://creandotumatrix.com), la firma de operación de
> IA del "Día 2". Cada línea incluye: qué aporta aria, qué recibe el cliente,
> y la receta concreta.

---

## The stack / La pila

```
┌─────────────────────────────────────────────────────────────┐
│  Turbo Rig   — governs the coding agents (gate, merges)     │
│  Turbo Flow  — gates AI-written code (fail-closed)          │
│  turbo-brain-v2 — the agents' own memory (MCP, read-only)   │
│  aria (this repo) — the COMPANY's knowledge:                │
│      documents → embeddings → grounded answers over an API  │
└─────────────────────────────────────────────────────────────┘
```

Agents need two kinds of memory: their own (sessions) and the company's
(handbooks, runbooks, policies, delivery records). aria is the second kind.
It is deliberately boring to operate — API-key auth, rate limits, structured
JSON logs, metrics — because that operability floor is itself part of the
offer.

Los agentes necesitan dos memorias: la suya (sesiones) y la de la empresa
(manuales, runbooks, políticas, registro de entregas). aria es la segunda. Es
deliberadamente aburrida de operar — auth por API key, rate limits, logs JSON
estructurados, métricas — porque ese piso de operabilidad es en sí parte de
la oferta.

### aria's operability floor (as shipped) / Piso de operabilidad de aria

| Capability / Capacidad | Where / Dónde |
|---|---|
| API-key auth (`X-API-Key` on `/ingest`, `/query`, `/query/stream`, `/metrics`; `/health` public) | `API_KEY`, `app/auth.py` |
| Per-key rate limiting (60 req/min default) | `RATE_LIMIT_RPM`, `app/rate_limit.py` |
| Structured JSON logging, per-request latency | `LOG_LEVEL`, `app/logging_config.py` |
| Live metrics: queries, documents, avg latency, fallbacks | `GET /metrics` |
| CORS allowlist, credentials only for listed origins | `CORS_ORIGINS` |
| Ingest deduplication persisted across restarts | `DEDUP_STORE_PATH` |
| Provider fallback (primary LLM → fallback LLM) | `LLM_MODEL`, `FALLBACK_LLM_MODEL` |
| Streaming answers over SSE | `GET /query/stream` |
| CI on every push/PR (full pytest suite) | `.github/workflows/ci.yml` |

---

## Línea 1 · Operación del Día 2 / Track 1 · Day 2 operations

**EN — what aria provides.** The firm's audits ask: is it up, is it auditable,
is it used? aria is a working reference of what "yes" looks like — every
checkbox above is implemented and tested in this repo, not aspirational. When
a client's knowledge tooling is the gap found in an audit, aria is the
fixed-scope replacement: Dockerfile and compose file included, health endpoint
for the load balancer, metrics endpoint for the dashboard, logs that ship to
any aggregator as-is.

**ES — qué aporta aria.** Las auditorías de la firma preguntan: ¿está arriba,
es auditable, se usa? aria es una referencia viva de cómo se ve el "sí" — cada
casilla de la tabla está implementada y probada en este repo. Cuando la
brecha detectada en una auditoría es la herramienta de conocimiento, aria es
el reemplazo de alcance fijo: Dockerfile y compose incluidos, health endpoint
para el balanceador, métricas para el tablero, logs que llegan listos a
cualquier agregador.

**Recipe / Receta** (compose passes the container its environment from `.env` —
`env_file` in `docker-compose.yml` — so set keys there, not in the shell)

```bash
# Deploy with the Day 2 floor on / Receta de despliegue con el piso Día 2
cp .env.example .env
openssl rand -hex 24        # generate the API key / genera la clave
# then edit .env: API_KEY=<that key>  CORS_ORIGINS=https://intranet.cliente.com
docker compose up -d        # compose ships its own /health healthcheck
curl -s http://localhost:8000/health          # probe for the LB / sonda
API_KEY="$(grep -m1 '^API_KEY=' .env | cut -d= -f2)"
curl -s -H "X-API-Key: $API_KEY" http://localhost:8000/metrics
```

---

## Línea 2 · Transición agéntica / Track 2 · The agentic transition

**EN — what aria provides.** The transition offer teaches teams to direct
agents with discipline: parallel agents, isolated worktrees, gated review,
human judgment on every merge. The recurring failure in cohorts is agents
answering from training data instead of the team's actual practices. The fix
is the **onboarding corpus**: the team's runbooks, conventions, and decision
records ingested once — then every agent (and every new hire) queries the
same source of truth. Ingest is idempotent (dedup on `source` + content), so
re-ingesting after a doc changes is a safe cron, not a migration.

**ES — qué aporta aria.** La oferta de transición enseña a los equipos a
dirigir agentes con disciplina. El fallo recurrente en las cohortes son los
agentes que responden desde sus datos de entrenamiento en vez de las
prácticas reales del equipo. La solución es el **corpus de onboarding**:
runbooks, convenciones y decisiones del equipo ingestados una vez — y desde
entonces cada agente (y cada persona nueva) consulta la misma fuente. El
ingest es idempotente (dedup por `source` + contenido), así que re-ingestar
tras un cambio de documento es un cron seguro, no una migración.

**Recipe / Receta**

```bash
# 1. Ingest the team's knowledge / Ingerir el conocimiento del equipo
curl -X POST http://localhost:8000/ingest \
  -H "X-API-Key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"content": "# Deploy runbook\n\n...", "source": "runbooks/deploy.md"}'

# 2. Agents (and humans) ask in plain language / Agentes y personas preguntan en lenguaje natural
curl -X POST http://localhost:8000/query \
  -H "X-API-Key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"question": "¿Cuál es el proceso de rollback?", "top_k": 5}'

# 3. Answers streamed token by token / Respuestas en vivo, token a token
curl -N "http://localhost:8000/query/stream?question=como+hacemos+release" \
  -H "X-API-Key: $API_KEY"
```

For LATAM deliveries, answer prompts are edited to Spanish in
`config/prompts.yaml` (plain YAML; the default templates ship in English);
chunk size and retrieval depth tune per corpus with
`CHUNK_SIZE` / `CHUNK_OVERLAP` / `TOP_K`.

---

## Línea 3 · Construcción a medida / Track 3 · Custom builds

**EN — what aria provides.** The canonical "internal tool you can't buy off
the shelf" is a private company-knowledge assistant: the client's documents,
the client's embedding model, the client's LLM provider, inside the client's
perimeter. aria is the reference build for that delivery — fork, swap the
client-specific parts, ship. The swap points are configuration, not code:

| Swap point / Punto de cambio | Knob / Perilla |
|---|---|
| Embedding model | `EMBEDDING_MODEL` (default `all-MiniLM-L6-v2`) |
| Primary / fallback LLM | `LLM_MODEL`, `FALLBACK_LLM_MODEL` (OpenAI → Gemini) |
| Retrieval depth & chunking | `TOP_K`, `CHUNK_SIZE`, `CHUNK_OVERLAP` |
| Tone, language, answer shape | `config/prompts.yaml` |
| Vector store persistence | `CHROMA_PERSIST_DIR` |
| Streaming timeout | `STREAM_TIMEOUT_S` |

Mapped to the firm's four-week delivery model / Mapeado al modelo de entrega
de cuatro semanas de la firma:

1. **Week 1 — Idea → plan.** Corpus inventory: which documents, which
   questions must be answerable, how success is measured.
2. **Week 2 — Build with agents.** aria configured to the client's providers
   and perimeter; corpus ingested; cohort queries validated against the week-1
   question list.
3. **Week 3 — Guardrails.** `API_KEY` issuance, `RATE_LIMIT_RPM` sizing,
   `CORS_ORIGINS` allowlist, log shipping, `/metrics` on the client dashboard.
4. **Week 4 — Live, with evidence.** Public health endpoint, metrics
   snapshot, and a delivery record entry: what shipped, when, where it lives.

---

## Honest limits / Límites honestos

- Retrieval quality carries no published benchmark (`eval/` ships a
  30-question benchmark set, not a score); per-delivery, quality is validated
  against the client's own question list. / La calidad de recuperación no
  publica benchmark (`eval/` trae un set de 30 preguntas, no un puntaje); por
  entrega se valida contra la lista de preguntas del cliente.
- Single-node ChromaDB. Right for a company knowledge base; not a
  multi-tenant SaaS backend. / ChromaDB de un solo nodo. Sirve para base de
  conocimiento de empresa; no es backend multi-tenant.
- Ingestion reads Markdown and plain text (`app/ingestion/`). PDFs/Office
  docs are converted to Markdown upstream of `/ingest`. / La ingesta lee
  Markdown y texto plano; PDFs y Office se convierten a Markdown antes de
  `/ingest`.

---

*aria is part of the [Adventure Wave Labs](https://www.adventurewavelabs.space)
stack — the research lab behind [Turbo Rig](https://turbo-rig.com) and
[Turbo Flow](https://www.turboflow.online).*
