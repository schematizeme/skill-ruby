# Entrega: Templates, Flags, IA Assistida, DoD, Evolução e Índice

> Parte da skill **schematize-ruby**. Continuação de `operacao.md` (numeração de seções **preservada**, §29+): templates, feature flags, uso de IA assistida, Definition of Done, evolução e índice de funcionalidades. Cross-refs por número de seção continuam válidos.

---

## 29. Templates

```
/templates
├── README.md
├── ADR.md
├── TASK.md
├── PR_TEMPLATE.md
├── ISSUE_TEMPLATE.md
├── RUNBOOK.md
├── INDEX_GLOBAL.md       # índice global da aplicação (§39)
├── INDEX_FUNCTIONS.md    # índice de microfunções (§39)
└── OPENAPI_TEMPLATE.yaml
```

README mínimo: o que é, como rodar (`bundle install` + `bin/` scripts + `bundle exec puma`/`rails s`), como testar (`bundle exec rspec`), como deployar, dependências, observabilidade, oncall, runbook.

---

---

## 31. Feature Flags

Obrigatório para features críticas, migrações e rollouts graduais.
Capacidades: rollout por % de tráfego, segmentação por tenant/usuário, kill switch, expiração de flag.
Sugestões: Flipper, Unleash, OpenFeature, GrowthBook.

---

---

## 34. Uso de IA Assistida

**MUST**
- Código gerado por IA passa pelo mesmo PR review humano que qualquer outro.
- Autor humano é responsável: assina o commit, entende o código, mantém.
- Saídas de IA não substituem ADR.
- **IA opera sob a §37 (anti-padrões vetados) integralmente.** Gerar código que viola um item VETADO é defeito, não estilo — rejeita no review.
- **IA gera o archive (§28) junto com o código**, na mesma entrega.

**MUST NOT / VETADO**
- Colar trecho gerado sem ler.
- Aceitar gems sugeridas sem verificar nome (typosquatting no RubyGems é real — §37).
- Submeter código que não passa em `bundle exec rake ci` (RuboCop + brakeman + bundler-audit + rspec).
- Aceitar "solução rápida" da IA que burla um piso de segurança da §37.

**SHOULD**
- Prompt e contexto relevantes registrados no chat archive quando a decisão for não trivial.
- Verificar licença de snippets longos sugeridos.

### 34.1 Handoff de contexto em sessões longas

Sessões longas de agente degradam quando o contexto enche: o modelo "esquece" decisões e a compactação automática resume de forma lossy. Para não perder estado, o handoff é **proativo e arquivado**, não reativo.

**MUST**
- Definir um **limite de handoff** (tokens) abaixo do teto da janela — cedo o suficiente pra sobrar espaço pra escrever o resumo. Sugestão: ~25% da janela (ex.: 250k numa janela de 1M).
- Ao cruzar o limite, **antes de qualquer compactação**, gerar dois artefatos em `<project>_archive/context/`:
  - `<YYYY-MM-DD-HH-MM-SS>-context.md` — estado do projeto, decisões tomadas, arquivos tocados, onde parou.
  - `<YYYY-MM-DD-HH-MM-SS>-checklist.md` — **feito vs. em aberto**.
- Só então compactar/limpar o contexto. O handoff é archive obrigatório (§28) — armazenar **sempre** em `<project>_archive`.
- **Rede de segurança determinística:** um backup do estado é capturado automaticamente antes de toda compactação (manual ou automática), independente de o agente ter lembrado de gerar os MDs acima.

**SHOULD**
- O limite é configurável por ambiente/projeto (ex.: env var), não hardcoded.
- O resumo de contexto preserva o que **você** escolhe (foco na tarefa corrente), não o que a compactação automática adivinha.

> Compactação automática é rede, não plano. Em sessão longa, o handoff arquivado vem antes do teto — quem controla o que sobrevive é você, não o resumo lossy.

---

---

## 35. Definition of Done

Uma task está pronta quando, cumulativamente:

- [ ] Testes passam (`bundle exec rspec` — unit + integration), cobertura nos mínimos (**SimpleCov** no piso configurado)
- [ ] Caminhos críticos com testes explícitos
- [ ] **Teste emulado por IA (`simulated`, a `schematize-qa` (smoke e matriz simulated, `references/categorias.md` secoes 5 e 10)) executado — 100% das rotas do inventário acessíveis pra quem deve e bloqueadas pra quem não deve; rota fantasma/morta = bloqueio**
- [ ] **Pentest de entrada limpo: sem `500`, sem coerção de tipo, sem eco não-escapado, sem vazamento cross-tenant; `brakeman` sem achado alto (a `schematize-qa` (smoke e matriz simulated, `references/categorias.md` secoes 5 e 10), a `schematize-pentest`)**
- [ ] **RuboCop** verde, `brakeman` e `bundler-audit` limpos
- [ ] **Nenhum item da §37 (anti-padrões vetados) presente no diff**
- [ ] **Arquivos `.rb` ≤ 750 linhas (~500 úteis + ~250 comentário); código útil > 300 linhas (~400 obs) flagueado e registrado como dívida (§6); toda classe/método público com doc-comment YARD de contexto — o quê + onde é usado**
- [ ] **Índice de funcionalidades atualizado no mesmo PR — global e microfunções (§39)**
- [ ] Observabilidade implementada (logs, métricas, traces, audit se aplicável)
- [ ] OpenAPI atualizada (se for API)
- [ ] Migration testada com rollback (`rails db:migrate`/`db:rollback`, `down` que funciona — se houver schema change)
- [ ] Documentação atualizada (README, ADR, runbook se aplicável)
- [ ] Smoke tests executados em staging **(com asserção de conteúdo e self-check anti verde-mentiroso — a `schematize-qa` (smoke e matriz simulated, `references/categorias.md` secoes 5 e 10))**
- [ ] CI verde (`bundle exec rake ci`), code review aprovado
- [ ] **Archive de chat/task gerado e commitado (§28) — gate rígido, não opcional**
- [ ] Feature flag configurada (se aplicável)
- [ ] CODEOWNERS aplicável revisou

- [ ] **Nenhum efeito externo real fora de `prd` (se o projeto envia e-mail/SMS/push/webhook/cobrança):** provider default = **sink**, **guard deny-by-default dentro do provider** (com teste que **vê a recusa**), **cap por execução** válido em TODOS os ambientes, e endereços só no **domínio de teste em rota nula**. Normativa: `schematize-engineering` → `references/efeitos-externos.md`; recorte desta linguagem em `references/iam.md` §3.1; anti-padrão §37 *"Disparar efeito externo REAL a partir de não-produção"* (citado **por título**, porque a numeração do §37 diverge entre skills)

> Os itens em negrito são **bloqueantes absolutos**: archive (§28), ausência de macaquice (§37), **nenhum efeito externo real fora de `prd`** (`schematize-engineering` → `references/efeitos-externos.md`), teste emulado por IA com rota 100% acessível (ver a `schematize-qa` (smoke e matriz simulated, `references/categorias.md` secoes 5 e 10)), e pentest de entrada limpo (ver a `schematize-pentest`). Faltando qualquer um, a task **não está pronta** — independente de todo o resto estar verde. Smoke verde não basta: tem que ser smoke que **prova** conteúdo, não só status.

---

---

## 36. Evolução

- Refactors incrementais. Big-bang rewrite exige ADR e plano de rollback.
- Toda migração de runtime/framework (ex.: Rails major, Sinatra→Rails, Ruby major) tem flag de coexistência.
- DDD e hexagonal podem ser adotados progressivamente — comece pelas bordas e pelos domínios mais complexos.

---

---

## 39. Índice de Funcionalidades (fonte da verdade viva)

O código diz **como** está agora; o índice diz **o que existe, onde mora e como se faz cada coisa** — e é tratado como **fonte da verdade do projeto**, consultado antes de criar algo (pra não duplicar) e atualizado a cada mudança. Índice que apodrece é pior que não ter; por isso ele é gerável/validável e tem gate na DoD.

**MUST — existência e localização**
- Todo projeto mantém o índice versionado em `<project>_archive/index/` (ou `/docs/index/`), em **dois níveis**:
  - **Índice global da aplicação** (`INDEX_GLOBAL.md`) — o mapa macro: repos/serviços/bounded contexts e como se comunicam; a relação de **pastas top-level** de cada repo e a responsabilidade de cada uma; **o que cada coisa faz** e o ponto de entrada de **como se faz** (link pro fluxo/use-case/runbook). É o "mapa do território".
  - **Índice de microfunções** (`INDEX_FUNCTIONS.md`, por serviço) — o catálogo fino: cada método/classe/módulo relevante → **o quê**, **onde é usado/previsto**, dependências e efeitos colaterais. Gerado a partir dos doc-comments YARD obrigatórios (§6).

**MUST — atualização e gate**
- Todo PR que **adiciona, remove ou move** funcionalidade atualiza o índice no mesmo PR. Índice desatualizado **trava o merge** (item da DoD, §35).
- O índice é **fonte da verdade**: ao planejar uma feature, consulte-o primeiro pra não reimplementar o que já existe (anti-duplicação — liga com DRY semântico, §1).
- Formato **machine-friendly** (markdown com tabelas, ou JSON/YAML que renderiza) pra permitir geração e validação automáticas — não prosa solta.

**MUST — completude (uma entrada por método, sem "relevante")**
- O índice de microfunções é **exaustivo**: **uma entrada por unidade chamável** — método, `def` público/privado, handler/controller action, job Sidekiq/ActiveJob, rake task, lambda nomeada, consumer — de **cada** serviço/repo do sistema, **pública e privada**. Não existe método "irrelevante": se está no código, está no índice. "Método relevante" **não é filtro** pra pular nada.
- **Invariante verificável (conte, não confie):** por serviço, `nº de entradas no índice == nº de métodos declarados no código`. O `/ruby-index` e o CI **contam as declarações** (`rg -n '^\s*def '` mais métodos definidos via `define_method`/`attr_*` quando expõem API, ou AST via `prism`/`parser`) e **reprovam** se o índice tiver **menos** entradas que métodos encontrados — listando os que faltam **pelo nome**. Índice com 90 linhas para 100+ métodos é **falha dura**, não aviso. O mapa não "resume" o sistema; ele **enumera** o sistema.
- **Cobertura total:** o índice **global** lista **cada** microserviço/repo (nenhum de fora); cada microserviço tem seu índice de métodos **completo**. Um sistema de N serviços com M métodos tem os N serviços mapeados e os M métodos indexados.

**MUST — o mapa é um GRAFO, não uma lista**
- O índice/MAPA carrega um **grafo textual** de dependências, navegável em dois níveis:
  - **Grafo de serviços** (cross-service): nós = microserviços; arestas `A → B` rotuladas pelo **contrato** (rota/evento/fila/tópico Sidekiq). Quem chama/notifica quem no sistema inteiro.
  - **Grafo de chamadas** (intra-service): por método, **quem ele chama** (out) e **quem o chama** (in) — adjacência `chamador → chamada`. Percorre-se de um ponto de entrada até a saída, e vê-se o **raio de impacto** de qualquer método.
- **Formato:** bloco **Mermaid** (`flowchart`) — textual **e** renderiza no GitHub/markdown — **mais** a adjacência em lista/tabela (pra diff, busca e grep). O Mermaid é o desenho; a adjacência é a fonte pesquisável. Ambos gerados pelo `/ruby-index`.

**SHOULD — geração assistida**
- O índice de microfunções é **gerado por script** que varre os doc-comments YARD padronizados (§6) e monta a tabela `método → o quê → onde → arquivo:linha`. CI compara o índice commitado com o gerado; divergência aponta índice ou comentário desatualizado.
- Cada entrada linka pro arquivo/linha de origem.
- Índice global revisado em cada mudança arquitetural (junto com o ADR, §27).

**Conteúdo mínimo**

`INDEX_GLOBAL.md`: lista de repos/serviços com 1 linha de propósito cada; por repo, árvore de pastas top-level com responsabilidade; mapa de comunicação (quem chama quem, quais eventos/contratos/filas Sidekiq); links pra OpenAPI, SLO, runbook.

`INDEX_FUNCTIONS.md` (por serviço, **exaustivo — uma linha por método**): `método | o quê | de onde vem → pra onde vai | chama (out) | é chamado por (in) | efeitos | arquivo:linha`; **nº de linhas == nº de métodos do serviço**. Acompanha o **grafo de chamadas** (Mermaid + adjacência). O `INDEX_GLOBAL.md` inclui o **grafo de serviços** (Mermaid) com **todos** os microserviços e seus contratos.

> O índice responde "isso já existe? onde? como faço X?" sem precisar reler o código. Se a resposta exige caçar no código, o índice falhou — ou está desatualizado, e isso é bug.

---

---

## Anexo A — Versões Correntes

> Atualizado independentemente do documento principal. Revisão trimestral.

| Stack | Versão alvo (2026-05) |
|---|---|
| Ruby | 3.4 |
| Rails | 8.x (ou Sinatra/Hanami conforme o contexto) |
| Node.js | 24 LTS (frontend: Next.js, Astro, etc. — §3) |
| PostgreSQL | 16+ |
| Redis | 7+ (fila Sidekiq) |
| Kubernetes | 1.30+ |
| OpenAPI | 3.1 |
| OpenTelemetry | 1.x (estável) |

Mudanças de versão major (Ruby/Rails) exigem ADR.

---

---

## Anexo B — Glossário Mínimo

- **Bounded Context** — fronteira explícita dentro da qual um modelo de domínio é consistente.
- **Monólito distribuído** — serviços fisicamente separados mas acoplados por banco compartilhado, gem de domínio compartilhada ou cadeia síncrona sem fronteira. O pior dos dois mundos. Proibido (§2).
- **BFF (Backend for Frontend)** — camada server-side que serve um frontend específico e mantém os segredos fora do browser (§38).
- **Outbox Pattern** — gravar evento em tabela no mesmo commit do dado de negócio; publicador assíncrono (job Sidekiq) lê a tabela e publica no broker. Garante consistência sem dual-write.
- **DLQ** — dead letter queue, fila de mensagens que falharam após retries (fila morta do Sidekiq).
- **Anti-Corruption Layer** — adapter que isola seu domínio do modelo externo.
- **SLO** — service level objective, alvo mensurável de qualidade (ex: 99.9% das requests < 300ms em 30 dias).
- **Error Budget** — quanto você pode falhar dentro do SLO antes de freezar features.
- **Blameless Postmortem** — análise de incidente focada em sistema/processo, não em culpa individual.
- **CSPRNG** — gerador pseudoaleatório criptograficamente seguro. Obrigatório para tokens, ids de sessão e segredos (`SecureRandom`, nunca `rand`/`Random` — §14).
- **Macaquice** — atalho que parece entregar mais rápido e entrega vulnerabilidade ou dívida. Catalogadas e vetadas na §37.

---
