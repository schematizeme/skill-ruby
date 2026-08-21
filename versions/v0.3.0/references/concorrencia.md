# Concorrência e Paralelismo (Ruby)

> Parte da skill **schematize-ruby**. O erro mais caro de concorrência em Ruby não
> é de sintaxe — é **confundir concorrência com paralelismo**: achar que 10 threads
> aceleram um cálculo CPU-bound, ou compartilhar estado mutável sem lock e ver
> corrupção sob carga. Esta reference é o piso. Liga com `stack-versoes.md`
> (versão do Ruby/Puma/Sidekiq), `arquitetura.md` (§3 — quando NÃO usar Ruby) e
> `dados-eventos.md` (jobs, resiliência, idempotência).

## Índice
- C1. A GVL — conheça antes de escalar
- C2. Threads e estado compartilhado
- C3. Paralelismo real: processos (Puma, fork)
- C4. Ractor (experimental)
- C5. Async / Fibers e I/O de alta concorrência
- C6. Jobs — Solid Queue (default) e Sidekiq
- C7. Guia de decisão I/O-bound × CPU-bound

---

## C1. A GVL (Global VM Lock) — conheça antes de escalar

No CRuby/MRI existe a **GVL**: **só uma thread executa bytecode Ruby por vez**. Threads
dão **concorrência para trabalho I/O-bound** (a GVL é liberada em I/O bloqueante e em
muitas C-extensions), **não paralelismo de CPU**.

**PISO**
- **Conheça a GVL antes de escolher o modelo de concorrência.** Paralelismo real de
  CPU no MRI exige **PROCESSOS** ou **Ractors** — **nunca só threads**. Empilhar
  threads num cálculo CPU-bound não acelera nada: elas se revezam na mesma GVL.

**MUST**
- Classificar o trabalho como **I/O-bound** (rede, DB, disco, chamada externa) ou
  **CPU-bound** (cálculo, serialização pesada, compressão, cripto) **antes** de decidir.
- Se for CPU-bound e virar gargalo, escalar por processo/Ractor ou **extrair para
  Go/Rust** (§3 de `arquitetura.md`), não empurrar mais thread.

> JRuby/TruffleRuby **não** têm GVL (têm paralelismo de thread real), mas o default
> da casa é MRI — assuma GVL a menos que o ADR diga o contrário.

---

## C2. Threads e estado compartilhado

**MUST**
- Threads (`Thread`, `concurrent-ruby`) valem para **I/O-bound**: várias chamadas
  externas/consultas em voo enquanto cada uma aguarda rede/DB.
- **Toda estrutura compartilhada e mutável entre threads é protegida** por `Mutex`
  (ou usa estruturas de `concurrent-ruby`: `Concurrent::Map`, `Concurrent::Array`).

**VETADO**
- **Estado compartilhado mutável sem lock.** `Hash`/`Array`/variável de classe
  acessados por várias threads sem `Mutex` → race, corrupção, leitura suja. "Nunca
  deu problema" é sorte da GVL trocando de contexto num ponto conveniente — sob
  carga, quebra.
- Variável global/de classe usada como cache mutável em app multithread (Puma)
  sem sincronização.

**SHOULD**
- Preferir **imutabilidade e passagem de mensagem** (fila) a compartilhar memória —
  elimina a classe inteira de bug de corrida.
- `Thread.new` solto sem supervisão é fumaça: use um pool (`concurrent-ruby`
  `ThreadPoolExecutor`) com limite, e **cheque a exceção** de cada thread (thread que
  levanta e morre em silêncio some sem rastro).

---

## C3. Paralelismo real: processos (Puma cluster, fork)

O paralelismo de CPU de verdade no MRI vem de **múltiplos processos**.

**MUST**
- Servir HTTP com **Puma em modo cluster**: `workers` = processos (forkam o master),
  cada worker com seu pool de `threads`. Workers dão paralelismo (CPU entre cores);
  threads dentro do worker dão concorrência de I/O.
- Dimensionar `workers` ~ nº de cores; `threads` conforme o perfil I/O da app.
- **Pool de conexões do DB ≥ threads por worker** — senão thread fica esperando
  conexão e a concorrência é fictícia (liga com C6).

**SHOULD**
- Aproveitar **Copy-on-Write**: `preload_app!` no Puma pra compartilhar memória entre
  workers no fork.
- Vigiar **memory bloat / fragmentação**: alocador **jemalloc** e
  `MALLOC_ARENA_MAX=2` reduzem o inchaço típico de processo Ruby de vida longa.
- `Process.fork` para paralelizar tarefa CPU-bound pontual (cada filho é isolado);
  colher o status de cada filho (`Process.wait`) — filho zumbi/erro engolido é bug.

---

## C4. Ractor (paralelismo real, memória isolada)

`Ractor` (Ruby 3+) dá **paralelismo de CPU real dentro do processo**: cada Ractor tem
seus objetos isolados e só troca dados imutáveis/`shareable` ou por cópia/move.

**Honestidade obrigatória: Ractor é EXPERIMENTAL.**
- A API ainda é **instável** e muitas gems (inclusive partes da stdlib e do ecossistema
  Rails) **não são ractor-safe** — usar objeto não-shareable estoura em runtime.

**MUST**
- Não usar Ractor como default. Adotar **só com ADR** justificando o ganho, com
  **cobertura de teste dedicada** e ciente de que a superfície pode quebrar entre
  minors do Ruby.

**SHOULD**
- Antes de Ractor, medir se **processo/Puma worker** já resolve — quase sempre resolve,
  com ecossistema maduro. Ractor é aposta, não ferramenta de prateleira.

---

## C5. Async / Fibers e I/O de alta concorrência

Para **muita** concorrência de I/O sem o custo de memória de uma thread por conexão,
a gem **`async`** (event loop sobre **fibers**) e o `Fiber.scheduler` (Ruby 3+)
suspendem a fiber no `.await`/I/O e retomam outra — milhares de operações I/O em voo
num punhado de threads.

**MUST**
- Backpressure/limite explícito no fan-out: `Async::Semaphore` (ou `Async::Barrier`)
  pra não disparar 10k requisições de uma vez contra um upstream. Concorrência
  ilimitada é OOM/derrubada-de-dependência esperando acontecer.
- Toda operação de I/O tem **timeout** (`Async::Task#with_timeout`) — nada aguarda
  pra sempre.

**VETADO**
- Chamada **bloqueante** (driver síncrono que não cede ao scheduler, CPU pesado)
  dentro do event loop — trava a fiber e mata a concorrência de todas as outras.
  Trabalho CPU-bound sai do loop (processo/Ractor).

**SHOULD**
- `async` brilha em cliente de agregação/gateway com muitas chamadas externas;
  não é bala de prata pra CPU-bound (a GVL continua lá).

---

## C6. Jobs — Solid Queue (default) e Sidekiq

**Solid Queue é o default da casa** — é o adapter de Active Job do Rails 8 e roda **no banco que
você já tem**, sem Redis no caminho. Isso importa mais do que parece: com Sidekiq, a fila fica num
Redis **fora da transação** do seu dado, e é dali que nasce o job clássico que roda antes do commit
(ou depois de um rollback) e processa um registro que não existe. Com Solid Queue o enfileiramento é
**uma linha no mesmo Postgres**, então ele entra na **mesma transação** da escrita — o outbox de
graça. Menos uma peça de infra, menos um estado para reconciliar.

**Sidekiq continua legítimo** e é a escolha certa em dois casos: já existe e funciona (não se troca
fila em produção por moda), ou o volume/latência exige o que o Redis dá (throughput muito alto,
latência de enfileiramento em milissegundos, ou o ecossistema Pro/Enterprise). Trocar em qualquer
direção é **ADR**.

**MUST** (valem para os dois)
- Job é **I/O-bound e idempotente** (§19): ele pode rodar 2x (retry, crash entre efeito e ack) —
  reexecutar não pode duplicar cobrança nem estado.
- **Enfileire depois do commit**, nunca no meio da transação — com Solid Queue isso é natural
  (mesma transação); com Sidekiq é `after_commit`, e esquecer disso é o bug de fila mais comum
  do Rails.
- **Pool de conexões do DB ≥ concorrência do worker** — senão as threads brigam por
  conexão e travam (mesmo piso do Puma, C3). Com Solid Queue conte também as conexões que a
  **própria fila** usa para fazer polling.
- **Retry com limite e backoff**; job que estoura o limite vai pra dead set e
  **alerta** — não some (liga com `observabilidade.md`).

**SHOULD**
- CPU-bound pesado **não** vai num worker de alta concorrência (as threads se
  serializam na GVL e afogam o worker): use **poucas** threads, uma fila dedicada, ou
  um **processo à parte** (ou extraia pra Go/Rust).
- Estado compartilhado entre jobs (memoização a nível de processo) segue a regra de
  C2 — protegido ou evitado.

---

## C7. Guia de decisão — I/O-bound × CPU-bound

| Perfil do trabalho | Ferramenta | Observação |
|---|---|---|
| I/O-bound, concorrência moderada | **Threads** (Puma threads, `concurrent-ruby`) | GVL liberada no I/O; pool DB ≥ threads |
| I/O-bound, altíssima concorrência | **`async` / Fibers** | leve; `Async::Semaphore` p/ backpressure |
| I/O-bound assíncrono (fora do request) | **Solid Queue** (default) · Sidekiq quando o volume/latência pede | idempotente, retry+backoff, pool DB ≥ concorrência; enfileirar **depois do commit** |
| CPU-bound (paralelismo real) | **Processos** (Puma workers, `fork`) | 1 worker/core; CoW + jemalloc |
| CPU-bound in-process (aposta) | **Ractor** | EXPERIMENTAL — só com ADR + cobertura |
| CPU-bound que virou **gargalo** | **Extrair p/ Go/Rust** | §3 `arquitetura.md` — "quando NÃO usar Ruby" |

> Regra de bolso: **classifique I/O × CPU antes de escolher; threads dão concorrência
> de I/O e nunca paralelismo de CPU (culpa da GVL); paralelismo real é processo (ou
> Ractor experimental); toda fila/fan-out é limitado; e todo estado compartilhado
> mutável é protegido ou não existe.** Se o CPU-bound continua sendo gargalo depois
> disso, o problema não é a concorrência — é a linguagem: extraia pra Go/Rust.
