# Observabilidade, Healthchecks, Performance e FinOps


> **PONTEIRO, não cópia.** O padrão (OpenTelemetry, Collector, sinais correlacionados, SLO, alertas, dashboards
> como código, stack de referência) é da **`schematize-observability`** (`schematize-observability -> SKILL.md` e
> `references/`); o piso da base está em `schematize-engineering -> references/observabilidade.md`. Aqui fica
> **só o que muda em Ruby**: gems (OTel Ruby, `lograge`/`semantic_logger`, `yabeda`), `filter_parameters` do Rails,
> heap/GC por worker. **Onde divergir, a base e a `schematize-observability` mandam**
> (`SKILL.md` §"Precedência e herança"); a numeração salta porque o número é o da base.

## 16. Observabilidade

- **Instrumentação:** **OpenTelemetry Ruby** (`opentelemetry-sdk` + `opentelemetry-exporter-otlp` + `opentelemetry-instrumentation-all`) — traces, métricas e logs correlacionados por `trace_id`; exemplars ligando métrica↔trace. Auto-instrumentação de Rails/Rack, Sidekiq, `pg`/ActiveRecord e Redis liga sozinha.

- **Backends e stack:** `schematize-observability -> references/stack-referencia.md`. Profiling contínuo: `schematize-observability -> references/amostragem-profiling.md`.

### 16.1 Logs

- **JSON estruturado, uma linha por request:** **`lograge`** (Rails) ou **`semantic_logger`**. **VETADO** `puts`/`p`/`Logger` texto solto em produção (§37).

- `trace_id`, `request_id`/`correlation_id`, `tenant_id` (se aplicável) em toda request — correlacionados com o trace do OTel.

- **Proibido logar:** senhas, tokens, JWT, PII (CPF, email, telefone), dados financeiros, payloads de pagamento. Use `config.filter_parameters` do Rails; mascaramento obrigatório.

### 16.2 Métricas

- **RED por endpoint/controller/job:** Rate, Errors, Duration (histograma p50/p95/p99) — via `yabeda` + `yabeda-prometheus` (`/metrics`) ou exporter Prometheus nativo.

### 16.3 Tracing

- Toda chamada externa, fila (Sidekiq), banco e fluxo crítico instrumentados (auto-instrumentation cobre; span custom no domínio).

### 16.5 Business Observability

- Métricas de negócio expostas (pedidos/min, conversão, churn, etc — `yabeda` custom).

## 17. Healthchecks

Contrato liveness/readiness: `schematize-observability -> references/healthchecks.md`. Em Rails, `/ready` checa DB, Redis e Sidekiq (pronto pra tráfego); liveness não toca dependência.

## 30. Performance — Metas Padrão

| Métrica | Alvo |
|---|---|
| API p95 | < 300 ms |
| API p99 | < 1 s |
| Startup (boot Rails) | < 20 s |
| Imagem Docker | < 400 MB (`ruby:<versão>-slim` + multi-stage) |

**Atenção Ruby (§37):** N+1 (detectar com `bullet`), **memory bloat / crescimento de heap** por worker (fork do Puma/Sidekiq copia memória — monitorar RSS, considerar `jemalloc`, `MALLOC_ARENA_MAX`), pressão de GC. APM (OTel + Tempo, ou Skylight/AppSignal quando justificado). YJIT ligado em série suportada (§`stack-versoes.md`).

## 33. FinOps — Gestão de Custos

- Revisão periódica de overprovisioning (CPU/memória/réplicas). Em Ruby, **workers × threads vs memória** é o eixo central: cada worker Puma/Sidekiq é um processo com cópia de heap — dimensionar `WEB_CONCURRENCY`/threads contra a RAM real, não chutar.
