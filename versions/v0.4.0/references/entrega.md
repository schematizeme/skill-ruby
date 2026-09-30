# Entrega: Templates, Flags, IA Assistida, DoD, Evolução e Índice


> **PONTEIRO, não cópia.** A normativa deste tema é da base: **`schematize-engineering`** →
> `references/entrega.md`. Leia lá primeiro; aqui fica **só o que muda em Ruby**.
>
> **Onde este arquivo divergir da base, a BASE MANDA** (`SKILL.md` §"Precedência e herança").
> Em 2026-08-21 os blocos idênticos à base foram **podados mecanicamente** (`tools/podar-clone.mjs`),
> que é por que a numeração dos itens **salta**: o número é o da base, e o item que não aparece aqui
> é porque **não muda nesta linguagem** — procure-o lá. Manter a cópia era manter a próxima deriva
> (foi assim que o `argon2id-only` da casa virou "ou PBKDF2" numa skill e o rol de 6 linguagens
> virou "só Go e Rust" em três).

## 29. Templates

README mínimo: o que é, como rodar (`bundle install` + `bin/` scripts + `bundle exec puma`/`rails s`), como testar (`bundle exec rspec`), como deployar, dependências, observabilidade, oncall, runbook.

## 34. Uso de IA Assistida

- Aceitar gems sugeridas sem verificar nome (typosquatting no RubyGems é real — §37).

- Submeter código que não passa em `bundle exec rake ci` (RuboCop + brakeman + bundler-audit + rspec).

## 35. Definition of Done

- [ ] Testes passam (`bundle exec rspec` — unit + integration), cobertura nos mínimos (**SimpleCov** no piso configurado)

- [ ] **Teste emulado por IA (`simulated`, a `schematize-qa` (`references/categorias.md` §§5 e 10)) executado — 100% das rotas do inventário acessíveis pra quem deve e bloqueadas pra quem não deve; rota fantasma/morta = bloqueio**

- [ ] **Pentest de entrada limpo: sem `500`, sem coerção de tipo, sem eco não-escapado, sem vazamento cross-tenant; `brakeman` sem achado alto (a `schematize-qa` (`references/categorias.md` §§5 e 10), a `schematize-pentest`)**

- [ ] **RuboCop** verde, `brakeman` e `bundler-audit` limpos

- [ ] **Arquivos `.rb` ≤ 750 linhas (~500 úteis + ~250 comentário); código útil > 300 linhas (~400 obs) flagueado e registrado como dívida (§6); toda classe/método público com doc-comment YARD de contexto — o quê + onde é usado**

- [ ] Migration testada com rollback (`rails db:migrate`/`db:rollback`, `down` que funciona — se houver schema change)

- [ ] Smoke tests executados em staging **(com asserção de conteúdo e self-check anti verde-mentiroso — a `schematize-qa` (`references/categorias.md` §§5 e 10))**

- [ ] CI verde (`bundle exec rake ci`), code review aprovado

- [ ] **Nenhum efeito externo real fora de `prd` (se o projeto envia e-mail/SMS/push/webhook/cobrança):** provider default = **sink**, **guard deny-by-default dentro do provider** (com teste que **vê a recusa**), **cap por execução** válido em TODOS os ambientes, e endereços só no **domínio de teste em rota nula**. Normativa: `schematize-engineering` → `references/efeitos-externos.md`; recorte desta linguagem em `references/iam.md` §3.1; anti-padrão §37 *"Disparar efeito externo REAL a partir de não-produção"* (citado **por título**, porque a numeração do §37 diverge entre skills)

> Os itens em negrito são **bloqueantes absolutos**: archive (§28), ausência de macaquice (§37), **nenhum efeito externo real fora de `prd`** (`schematize-engineering` → `references/efeitos-externos.md`), teste emulado por IA com rota 100% acessível (ver a `schematize-qa`, `references/categorias.md` §§5 e 10), e pentest de entrada limpo (ver a `schematize-pentest`). Faltando qualquer um, a task **não está pronta** — independente de todo o resto estar verde. Smoke verde não basta: tem que ser smoke que **prova** conteúdo, não só status.

## 36. Evolução

- Toda migração de runtime/framework (ex.: Rails major, Sinatra→Rails, Ruby major) tem flag de coexistência.

## 39. Índice de Funcionalidades (fonte da verdade viva)

**MUST — existência e localização**

- Todo projeto mantém o índice versionado em `<project>_archive/index/` (ou `/docs/index/`), em **dois níveis**:
  - **Índice global da aplicação** (`INDEX_GLOBAL.md`) — o mapa macro: repos/serviços/bounded contexts e como se comunicam; a relação de **pastas top-level** de cada repo e a responsabilidade de cada uma; **o que cada coisa faz** e o ponto de entrada de **como se faz** (link pro fluxo/use-case/runbook). É o "mapa do território".
  - **Índice de microfunções** (`INDEX_FUNCTIONS.md`, por serviço) — o catálogo fino: cada método/classe/módulo relevante → **o quê**, **onde é usado/previsto**, dependências e efeitos colaterais. Gerado a partir dos doc-comments YARD obrigatórios (§6).

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

- O índice de microfunções é **gerado por script** que varre os doc-comments YARD padronizados (§6) e monta a tabela `método → o quê → onde → arquivo:linha`. CI compara o índice commitado com o gerado; divergência aponta índice ou comentário desatualizado.

- Cada entrada linka pro arquivo/linha de origem.

- Índice global revisado em cada mudança arquitetural (junto com o ADR, §27).

**Conteúdo mínimo**

`INDEX_GLOBAL.md`: lista de repos/serviços com 1 linha de propósito cada; por repo, árvore de pastas top-level com responsabilidade; mapa de comunicação (quem chama quem, quais eventos/contratos/filas Sidekiq); links pra OpenAPI, SLO, runbook.

`INDEX_FUNCTIONS.md` (por serviço, **exaustivo — uma linha por método**): `método | o quê | de onde vem → pra onde vai | chama (out) | é chamado por (in) | efeitos | arquivo:linha`; **nº de linhas == nº de métodos do serviço**. Acompanha o **grafo de chamadas** (Mermaid + adjacência). O `INDEX_GLOBAL.md` inclui o **grafo de serviços** (Mermaid) com **todos** os microserviços e seus contratos.

> O índice responde "isso já existe? onde? como faço X?" sem precisar reler o código. Se a resposta exige caçar no código, o índice falhou — ou está desatualizado, e isso é bug.

## Anexo A — Versões correntes → **`references/stack-versoes.md`**

> **Esta tabela foi REMOVIDA daqui.** Versão de terceiro é **fato com prazo de validade**, e ela
> vivia clonada em 8 `entrega.md` do catálogo — a mesma tabela, com a mesma data, apodrecendo em
> oito lugares ao mesmo tempo (ela ainda apontava uma versão de Go e uma de Kubernetes que **já estavam fora de suporte**). Fato volátil tem **um** lugar por skill: o **anexo volátil**, com **data de
> verificação** e cadência de revisão.
>
> **Onde está agora:** `references/stack-versoes.md` desta skill (versões de Ruby e do
> ferramental dela) e, para o que é de infraestrutura (Kubernetes, Postgres, Redis, OTel), a
> **`schematize-infra`**.
>
> **A regra que fica:** mudança de versão **major** exige ADR — isso não é volátil e continua aqui.
> E o lint do catálogo (`tools/lint.mjs`, regra `anexo-volatil`) **reprova** versão cravada no
> corpo normativo quando a skill tem anexo: é o detector que impede a próxima safra.

## Anexo B — Glossário Mínimo

- **Monólito distribuído** — serviços fisicamente separados mas acoplados por banco compartilhado, gem de domínio compartilhada ou cadeia síncrona sem fronteira. O pior dos dois mundos. Proibido (§2).

- **Outbox Pattern** — gravar evento em tabela no mesmo commit do dado de negócio; publicador assíncrono (job Sidekiq) lê a tabela e publica no broker. Garante consistência sem dual-write.

- **DLQ** — dead letter queue, fila de mensagens que falharam após retries (fila morta do Sidekiq).

- **CSPRNG** — gerador pseudoaleatório criptograficamente seguro. Obrigatório para tokens, ids de sessão e segredos (`SecureRandom`, nunca `rand`/`Random` — §14).
