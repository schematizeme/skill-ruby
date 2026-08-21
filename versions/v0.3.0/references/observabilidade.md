# Observabilidade, Healthchecks, Performance e FinOps


> **PONTEIRO, não cópia.** A normativa deste tema é da base: **`schematize-engineering`** →
> `references/observabilidade.md`. Leia lá primeiro; aqui fica **só o que muda em Ruby**.
>
> **Onde este arquivo divergir da base, a BASE MANDA** (`SKILL.md` §"Precedência e herança").
> Em 2026-08-21 os blocos idênticos à base foram **podados mecanicamente** (`tools/podar-clone.mjs`),
> e é por isso que a numeração **salta**: o número é o da base, e o item ausente aqui é o que **não
> muda nesta linguagem** — procure-o lá.

## 16. Observabilidade

- **Instrumentação:** **OpenTelemetry Ruby** (`opentelemetry-sdk` + `opentelemetry-exporter-otlp` + `opentelemetry-instrumentation-all`) — traces, métricas e logs correlacionados por `trace_id`; exemplars ligando métrica↔trace. Auto-instrumentação de Rails/Rack, Sidekiq, `pg`/ActiveRecord e Redis liga sozinha.

- **Backends:** Loki (logs), Tempo (traces), Prometheus (scrape/local) + **Mimir** (métricas long-term, HA, multi-tenant); **SHOULD** Pyroscope (profiling contínuo — `pyroscope`/`opentelemetry` profiling).

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

- `/ready` — readiness (dependências OK: DB, Redis, Sidekiq — pronto pra tráfego)

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
