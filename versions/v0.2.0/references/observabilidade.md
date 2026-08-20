# Observabilidade, Healthchecks, Performance e FinOps

> Parte da skill **schematize-ruby**. As referências cruzadas (§N) apontam para seções do corpo completo — todas presentes no conjunto de references desta skill.

## Índice
- 16. Observabilidade
- 17. Healthchecks
- 30. Performance — Metas Padrão
- 33. FinOps — Gestão de Custos

---

## 16. Observabilidade

**Stack obrigatória (LGTM+):** toda ferramenta/serviço criado ou atualizado — **inclusive o `<projeto>_ops`** — nasce com observabilidade **integrada de ponta a ponta**, nunca como extra depois:
- **Instrumentação:** **OpenTelemetry Ruby** (`opentelemetry-sdk` + `opentelemetry-exporter-otlp` + `opentelemetry-instrumentation-all`) — traces, métricas e logs correlacionados por `trace_id`; exemplars ligando métrica↔trace. Auto-instrumentação de Rails/Rack, Sidekiq, `pg`/ActiveRecord e Redis liga sozinha.
- **Coleta:** Grafana Alloy (coletor/agente OTel).
- **Backends:** Loki (logs), Tempo (traces), Prometheus (scrape/local) + **Mimir** (métricas long-term, HA, multi-tenant); **SHOULD** Pyroscope (profiling contínuo — `pyroscope`/`opentelemetry` profiling).
- **Visualização e alerta:** Grafana — dashboards e regras de alerta **versionados como código**, entregues junto do serviço.
- **Deploy:** **Helm chart** versionado por serviço — o repo entrega seu chart + dashboards + alertas com o código.
- Um serviço só é "pronto" se expõe `/metrics`, emite logs estruturados e traces, e sobe com dashboard + alertas + chart (§35). Avalie e acrescente o que fizer sentido (ex.: Grafana OnCall para on-call, k6 para carga, Beyla/eBPF para auto-instrumentação).

### 16.1 Logs

- **JSON estruturado, uma linha por request:** **`lograge`** (Rails) ou **`semantic_logger`**. **VETADO** `puts`/`p`/`Logger` texto solto em produção (§37).
- `trace_id`, `request_id`/`correlation_id`, `tenant_id` (se aplicável) em toda request — correlacionados com o trace do OTel.
- Níveis: DEBUG, INFO, WARN, ERROR.
- **Proibido logar:** senhas, tokens, JWT, PII (CPF, email, telefone), dados financeiros, payloads de pagamento. Use `config.filter_parameters` do Rails; mascaramento obrigatório.
- **VETADO** logar request/response inteiros, headers ou body cru "pra debugar" (§37). Logue campos específicos, mascarados.

### 16.2 Métricas

- **RED por endpoint/controller/job:** Rate, Errors, Duration (histograma p50/p95/p99) — via `yabeda` + `yabeda-prometheus` (`/metrics`) ou exporter Prometheus nativo.
- **USE para infra:** Utilization, Saturation, Errors.

### 16.3 Tracing

- Toda chamada externa, fila (Sidekiq), banco e fluxo crítico instrumentados (auto-instrumentation cobre; span custom no domínio).
- Propagação **W3C Trace Context**.

### 16.4 SLOs

- Cada serviço define SLI/SLO em `/docs/slo.md`.
- Error budget consumido → freeze de features até recuperação.

### 16.5 Business Observability

**SHOULD**
- Métricas de negócio expostas (pedidos/min, conversão, churn, etc — `yabeda` custom).
- Dashboards de negócio separados dos técnicos.
- KPIs principais instrumentados desde o dia 1.

### 16.6 Auditoria

**MUST**
- Operações sensíveis (mudança de permissão, transações financeiras, alteração de configuração, ações administrativas) registradas em **trilha de auditoria imutável**.
- Campos mínimos: `actor_id`, `tenant_id`, `action`, `resource`, `timestamp`, `ip`, `user_agent`, `result`.
- Retenção mínima conforme regulação aplicável.
- Storage append-only (não usar a mesma tabela de domínio).

---

---

## 17. Healthchecks

Endpoints obrigatórios:
- `/health` — liveness (processo está vivo)
- `/ready` — readiness (dependências OK: DB, Redis, Sidekiq — pronto pra tráfego)
- `/metrics` — Prometheus

---

---

## 30. Performance — Metas Padrão

| Métrica | Alvo |
|---|---|
| API p95 | < 300 ms |
| API p99 | < 1 s |
| Startup (boot Rails) | < 20 s |
| Imagem Docker | < 400 MB (`ruby:<versão>-slim` + multi-stage) |

Metas específicas sobrescrevem, registradas no `/docs/slo.md`.

**Atenção Ruby (§37):** N+1 (detectar com `bullet`), **memory bloat / crescimento de heap** por worker (fork do Puma/Sidekiq copia memória — monitorar RSS, considerar `jemalloc`, `MALLOC_ARENA_MAX`), pressão de GC. APM (OTel + Tempo, ou Skylight/AppSignal quando justificado). YJIT ligado em série suportada (§`stack-versoes.md`).

---

---

## 33. FinOps — Gestão de Custos

**SHOULD**
- Monitoramento de custo por serviço/squad/tenant.
- Budgets configurados com alertas de threshold.
- Revisão periódica de overprovisioning (CPU/memória/réplicas). Em Ruby, **workers × threads vs memória** é o eixo central: cada worker Puma/Sidekiq é um processo com cópia de heap — dimensionar `WEB_CONCURRENCY`/threads contra a RAM real, não chutar.
- Tags de billing consistentes em IaC.
- Custo por request rastreado em serviços de alto volume.

---

---
