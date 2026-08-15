# Eventos, Banco de Dados, Cache, APIs, Resiliência e Jobs

> Parte da skill **schematize-ruby**. As referências cruzadas (§N) apontam para seções do corpo completo — todas presentes no conjunto de references desta skill.

## Índice
- 9. Eventos e Mensageria
- 10. Banco de Dados
- 11. Cache
- 12. APIs
- 18. Resiliência
- 19. Jobs e Workers

---

## 9. Eventos e Mensageria

**MUST — produção e consumo**
- Eventos são imutáveis e versionados (`v1`, `v2`).
- Consumidores idempotentes (deduplicação por id de evento — chave única no banco ou `Sidekiq::Job` com `unique`).
- Suporte a replay.

**MUST — compatibilidade evolutiva**
- Novos campos são **opcionais**, com default seguro.
- Nunca remover ou renomear campo sem **nova versão** do evento.
- Consumidores **ignoram campos desconhecidos** (forward compatible).
- Coexistência de versões durante janela de migração documentada.

**MUST — entrega**
- **Transactional Outbox** para publicação de eventos (§18). Dual-write (banco + broker no mesmo fluxo, fora de transação) é **VETADO** (ver §37).
- DLQ / dead-set configurado para todo consumidor (Sidekiq mantém o dead-set nativo; DLQ explícita em Kafka/RabbitMQ).
- **Retry infinito proibido.** Política de retry explícita com limite e backoff.

**MUST — backpressure**
- Consumidores com limites explícitos de concorrência e prefetch (Sidekiq `concurrency`, `SIDEKIQ_CONCURRENCY`; prefetch no RabbitMQ; `max_poll_records` no Kafka).
- Throttling em produtores quando broker sinaliza pressão.

**Convenção de nome:** `<dominio>.<entidade>.<evento>` no passado.
Exemplos: `catalog.product.created`, `billing.invoice.paid`.

**Filas / brokers**

| Cenário | Stack |
|---|---|
| Jobs assíncronos, Redis já na casa, retry/dead-set nativos | **Sidekiq** (via ActiveJob ou direto) |
| Fila Postgres-backed (sem Redis dedicado), transação com o dado | **`good_job`** / **`solid_queue`** |
| Alto throughput, retenção, replay, streaming | **Kafka** (`rdkafka`/`karafka`) |
| Routing complexo | RabbitMQ (`bunny`) — justificar pelo custo operacional |

> Sidekiq é o padrão da casa quando há Redis. `good_job`/`solid_queue` quando o outbox e a fila devem viver na **mesma transação Postgres** do dado. Kafka/RabbitMQ exigem justificativa.

---

---

## 10. Banco de Dados

| Caso | Stack |
|---|---|
| Relacional | PostgreSQL |
| Cache | Redis |
| Busca textual | OpenSearch |
| Analytics / eventos | ClickHouse |

**ORM:** **ActiveRecord** (Rails) ou **Sequel** (Sinatra/Roda/serviço enxuto). Expor AR/Sequel cru na borda é **VETADO** — use serializer (§12).

**MUST**
- Migrations versionadas e **reversíveis**: `change` reversível **ou** par `up`/`down` — em ambos os casos o **rollback é testado** (`db:migrate` → `db:rollback` em CI), nunca só assumido. Migration irreversível declara `reversible`/`raise ActiveRecord::IrreversibleMigration` explícito. Automatizadas no deploy.
- Schema próprio por serviço.
- Sem joins cross-service no banco.
- **Timestamps em UTC, sempre** (`config.time_zone`/`ActiveRecord::Base.default_timezone = :utc`). Conversão de timezone apenas na borda (UI/API response).
- **IDs:** UUIDv7 ou ULID por padrão. IDs sequenciais apenas com justificativa (ex: ordenação natural exigida pelo domínio).
- **Toda query com input externo é parametrizada** (`where(id: params[:id])`, placeholders `where("x = ?", v)`). Concatenação/interpolação de string em SQL é **VETADA** (§37, liga `seguranca.md`).
- **N+1 é VETADO** em caminho quente: `includes`/`preload`/`eager_load` conforme o caso; detecção com **`bullet`** em dev/CI e revisão de query.
- **Connection pool dimensionado** (`pool` no `database.yml` ≥ threads do Puma/Sidekiq); pool menor que a concorrência é bug de saturação.

**SHOULD — alta escala**
- Separação leitura/escrita (réplicas — `connects_to`/`role: :reading`) quando volume justificar.
- Revisão periódica de índices; índice para toda FK e coluna de filtro/ordenação quente.
- Análise de query plan (`EXPLAIN ANALYZE`) em endpoints críticos.
- Particionamento para tabelas com crescimento previsível alto.

**Ferramentas sugeridas:** migrations nativas do Rails (`rails db:migrate`) / `sequel` migrations, `strong_migrations` (bloqueia migration perigosa em tabela grande), `bullet`, `pghero`.

---

---

## 11. Cache

**MUST**
- TTL **sempre explícito** (`expires_in:`). Sem TTL infinito sem ADR.
- Cache-aside (`Rails.cache.fetch(key, expires_in: ...) { ... }`) como padrão; mitigação de cache stampede (`race_condition_ttl`, lock, jitter).
- Fallback seguro em cache miss — degradação graciosa, nunca falha total.
- Invalidação documentada por chave.
- Resposta autenticada **sempre** tem a chave de cache segmentada por usuário e **tenant** (§37, §15).

**MUST NOT**
- Cache como source of truth.
- **Cache de dado sensível/pessoal sem escopo de tenant** e sem criptografia (§32).

**Stack:** Redis via `Rails.cache` (`redis_cache_store`); `solid_cache` (Postgres) como alternativa sem Redis dedicado.

---

---

## 12. APIs

**MUST**
- **OpenAPI 3.1** em `/docs/openapi.yaml` — fonte da verdade (gerar/validar com `rswag`/`committee`).
- Versionamento na URL: `/v1`, `/v2`.
- Validação de payload na borda: **strong parameters** (`params.require(...).permit(...)`) e/ou `dry-validation`/`dry-schema` (§38). `permit!`/`to_unsafe_h` em input de usuário é VETADO (§37).
- **Serializer explícito** na saída (`ActiveModel::Serializer`, `blueprinter`, `jbuilder`, `alba`) — nunca `render json: model` cru expondo colunas internas.
- Erro padrão (compatível com RFC 7807):

```json
{
  "error": {
    "code": "PRODUCT_NOT_FOUND",
    "message": "Product not found",
    "trace_id": "01HXYZ...",
    "details": []
  }
}
```

- Operações de escrita aceitam `Idempotency-Key` **e o implementam de fato** (persiste a chave e retorna a resposta original em replay; aceitar o header e ignorar é VETADO — §37).
- Quebra de contrato exige nova versão + janela de deprecação ≥ 90 dias com headers `Deprecation` e `Sunset`.
- Timestamps em **ISO-8601 com timezone explícito** (preferencialmente `Z` — `Time#iso8601`).

**MUST — paginação**
- **Cursor pagination** é o padrão.
- Offset apenas em listas pequenas (< 10k itens) e estáticas.
- Resposta inclui `next_cursor` e `has_more`.

**MUST — rate limiting**
- Rate limiting **distribuído** (`rack-attack` com store Redis, gateway, ou serviço dedicado).
- Chave por usuário, API key e tenant.
- Resposta 429 com header `Retry-After`.

**SHOULD**
- Contratos consumer-driven (Pact — `pact-ruby`) entre serviços.
- gRPC (`grpc`/`gruf`) para alta performance interna; HTTP/JSON para externa.

---

---

## 18. Resiliência

**MUST em chamadas externas:**
- Timeout explícito, nunca infinito — cliente HTTP **`faraday`** com `request.options.timeout`/`open_timeout` sempre setados; `Net::HTTP` cru sem timeout é bug.
- Retry com backoff exponencial + jitter (`faraday-retry`), idempotência garantida.
- Circuit breaker (`faraday` middleware + `circuitbox`/`stoplight`).
- Rate limiting na borda.
- Bulkhead em integrações críticas (pool/fila dedicada por dependência).

**MUST — independência e falha de dependência (resiliência por design):**
- **Serviço sobe e opera sozinho.** A ausência/queda de outra dependência (serviço, broker, cache, Redis) **nunca** impede o boot nem derruba este serviço — degrada (fallback, resposta parcial, enfileira e segue), não crasha em cascata (§2). "O `ledger` não sobe sem o `core`" é bug de acoplamento, não requisito.
- **Falha ao chamar/notificar outro serviço não se perde nem trava a cadeia.** Ao não conseguir notificar B, o serviço A **obrigatoriamente**: (1) **persiste o intento/dado em store durável** para retomar — **outbox no mesmo commit** (`good_job`/`solid_queue`/tabela de pendências na mesma transação AR), ou fila/Redis; nunca só em memória; (2) **loga com `trace_id`**; (3) **dispara alerta** (Grafana/alertmanager) pro ops/dev; (4) **retoma** com retry+backoff+jitter idempotente até o limite; estourou → **dead-set/DLQ** + escala pro humano. Nunca falha em silêncio, nunca perde o dado, nunca deixa a cadeia parada sem sinal.

---

---

## 19. Jobs e Workers

**MUST**
- Jobs **idempotentes e reentrantes** (pode executar 2x sem corromper — a lógica assume redelivery).
- Timeout obrigatório por job.
- Política de retry explícita com limite e **backoff** (Sidekiq: `sidekiq_options retry:`; ActiveJob: `retry_on`/`discard_on`).
- Progress/state persistido para jobs longos ou críticos.
- **Dead-set / DLQ** para jobs que falharem após max retries (Sidekiq dead-set nativo; monitorar e drenar).

**SHOULD**
- Jobs longos divisíveis em chunks com checkpoint.
- Cancelamento gracioso (respeitar shutdown/`TERM` do Sidekiq; não bloquear o quiet period).
- Argumentos de job **pequenos e serializáveis** (passar id, não o objeto AR inteiro — evita payload gordo e dado obsoleto).

---

---
