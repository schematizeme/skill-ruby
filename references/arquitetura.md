# Arquitetura, Camadas, Repositórios e Linguagens

> Parte da skill **schematize-ruby**. As referências cruzadas (§N) apontam para seções do corpo completo — todas presentes no conjunto de references desta skill.

## Índice
- 2. Estrutura de Repositórios
- 3. Linguagens
- 4. Arquitetura
- 5. Estrutura de Pastas
- 6. Complexidade e Tamanho
- 7. Dependências Internas e Shared Libraries
- 8. CQRS e Padrões de Aplicação

---

## 2. Estrutura de Repositórios

**MUST**
- Um repositório = uma aplicação ou um bounded context.
- Comunicação entre serviços via HTTP, gRPC, eventos ou mensageria — nunca via banco compartilhado.
- Cada serviço é dono do seu schema.
- **Nome do repositório:** `<projeto>_<contexto>[_<lang>]` em snake_case minúsculo. `<projeto>` = slug do produto/organização; `<contexto>` = a aplicação/bounded context daquele repo (`api`, `worker`, `front`, `backoffice`, `gateway`…); `_<lang>` é sufixo **opcional** pra desambiguar linguagem (`_rb` Ruby, `_go` Go, `_rs` Rust). Como um repo = um contexto, o nome espelha isso. Ex.: `loja_api_rb`, `loja_front`, `loja_worker_rb`.
- **Independência de runtime (cada serviço é entidade à parte):** todo serviço **sobe e opera sozinho**. A indisponibilidade de qualquer outro serviço **nunca** impede o boot nem derruba este — depender de outro serviço para *iniciar/funcionar* é VETADO (nada de "o `ledger` não sobe se o `core` estiver fora"). Dependente ausente vira **degradação graciosa** (fallback, resposta parcial, enfileira e segue), nunca crash em cascata. Como não perder o dado quando a chamada falha: `references/dados-eventos.md` (§18).
- **`<projeto>_ops` (control plane de desenvolvimento):** todo sistema multi-repo tem um repo **`<projeto>_ops`** — a ferramenta de operação do workspace, rodada por dev/agente e **fora do runtime do produto**. Faz bootstrap/instalação, update, manutenção, troubleshooting e roda os testes unitários/debug **através de todos os repos** (clona, sobe/para, migra, semeia e testa cada serviço). Não é microserviço nem é deployado com o produto; é essencial pra tocar um sistema de múltiplos repositórios. Como toda ferramenta, sobe com **observabilidade integrada** (Grafana/LGTM+, ver `references/observabilidade.md` §16).
- **Contenção no workspace (nunca sair da pasta do projeto):** o **diretório de projeto atual é o workspace**; toda aplicação/repo do sistema nasce e mora **dentro dele**. Vai criar uma aplicação nova? Crie uma **pasta pra ela dentro da pasta atual** (`./<projeto>_<contexto>/`) e trabalhe lá — **nunca** largue arquivos soltos no root pra depois **subir de diretório** (`cd ..`, `../`) e criar os outros repos fora. Num sistema multi-repo os repos são **irmãos dentro do mesmo workspace** (clonados ali pelo `<projeto>_ops`), não espalhados pela máquina. **VETADO** criar/ler/escrever fora do workspace: diretório-pai, `~`, `~/Documents`, `~/Downloads`, `/tmp` do usuário, Área de Trabalho. O agente **não sai da pasta do projeto** — nem pra vasculhar, nem pra criar — a menos que o usuário peça explicitamente.

**VETADO**
- **Aplicação monolítica que acopla múltiplos bounded contexts num só deploy/processo.** Não se cogita "começar monolito e quebrar depois" sem ADR explícito de plano e prazo de quebra. Misturar domínios de negócio "pra entregar rápido" é dívida disfarçada de produtividade. Em Rails isso é o default gravitacional — o `app/` cresce colando domínios; resistir a isso é obrigação, não zelo.
- **Monólito distribuído** — o pior dos dois mundos: serviços separados fisicamente, mas acoplados por banco compartilhado, shared lib de domínio (§7) ou chamadas síncronas em cascata sem fronteira. Tão proibido quanto o monólito clássico.
- **Big ball of mud** — código sem fronteira de contexto, onde tudo importa tudo.

**SHOULD**
- Evitar mais de um domínio de negócio no mesmo repositório.
- Leitura cross-service por réplica read-only só com ADR e contrato documentado.

**MAY**
- *Modular monolith* (módulos com fronteira de contexto rígida, schema separado, comunicação por interface interna) **somente com ADR** que justifique estágio do produto e contenha o plano de extração. É exceção registrada, não default. Nunca usar como atalho para colar domínios. Em Rails, a fronteira é **imposta por ferramenta** — `packwerk` (packs) com violações tratadas como erro de build (ver §5).

> Não existe "MVP monolítico que vira microserviço depois" sem o ADR que prova que o depois tem data. Sem isso, "depois" é "nunca", e "nunca" é um big ball of mud em produção. Um Rails app sem `packwerk` vira big ball of mud por gravidade, não por decisão.

---

---

## 3. Linguagens

**A casa não tem "a linguagem única"; tem o rol sancionado + guia de fit.** Detalhe agnóstico na base `schematize-engineering` (`references/linguagens.md`). Ruby é uma das sancionadas, e esta skill a especializa.

**Backend (serviço novo) — escolha UMA do rol, com ADR (§27) justificando o fit:**

| Linguagem | Skill | Sufixo de repo |
|---|---|---|
| **Go** | `schematize-go` | `_go` |
| **Rust** | `schematize-rust` | `_rs` |
| **Elixir** | `schematize-elixir` | `_ex` |
| **C#** (.NET) | `schematize-csharp` | `_cs` |
| **Zig** | `schematize-zig` | `_zig` |
| **Ruby** | `schematize-ruby` | `_rb` |

**Frontend — Node** (Next.js principal; Astro e outros consolidados), governado por `schematize-web`. É **só frontend** (o server-side do próprio front — route handlers/server actions/BFF — faz parte do frontend). Ruby **não** faz frontend: Hotwire/Turbo/ViewComponent de produto delega ao `schematize-web`; aqui o escopo é **API/backend/serviços/jobs/scripts**.

**Fora do rol (saída / legado — não reabrem):**
- **Node como serviço backend** e **PHP** **não recebem serviço novo.** São **legado**: ficam como estão até serem tocados, e migram para uma linguagem do rol **por funcionalidade do módulo** (~30% afetado → extrai o módulo; ~50% extraído → migra o resto; ajuste pontual não porta). Detalhe da saída em `schematize-node` (Node); **PHP migra sumariamente** — dívida ativa a ser zerada com ADR e plano, não "quando der".
- **Nova linguagem fora do rol** exige **ADR de exceção** aprovado — não se adota por gosto.

**MUST**
- Versão exata de Ruby/Rails em uso fica em `references/stack-versoes.md` (o "Anexo A" desta skill — inclui a janela de EOL). **Ruby EOL é VETADO** em produção.
- Não misturar linguagens dentro do **mesmo bounded context** sem ADR.
- **Ruby é SANCIONADA** — serviço novo em Ruby é **permitido por fit + ADR** (não é legado-only como Node backend). O ADR nomeia o fit que justifica a escolha.

**SHOULD**
- Escolher por **encaixe com o problema**, não por preferência. O default pragmático da casa é Go; Ruby entra quando o **fit** manda (ver §3.1).
- Frameworks são bem-vindos; abstrações mágicas não. Critério: consigo entender o stack trace e sei quem chamou o quê? Metaprogramação que esconde comportamento é veto (`references/anti-padroes.md`).

### 3.1 Ecossistema Ruby — quando (e quando NÃO) usar

**Regra de fit — Ruby encaixa quando:**
- **Prototipagem rápida** e validação de produto onde velocidade de iteração pesa mais que throughput.
- **Scripts / automação / glue** — tarefas de sistema, ETL leve, ferramentas internas.
- **DX de produto (Rails)** — CRUD rico, admin, back-office, painéis, onde o ecossistema (ActiveRecord, migrations, ActiveJob, generators) entrega valor cedo.
- **Manutenção e evolução de legado Ruby** já existente.

**Quando NÃO usar Ruby (o fit manda outra linguagem):**
- **Throughput/latência crítico ou CPU-bound pesado** (hot path com GVL sob pressão, cálculo numérico intenso) → **Go/Rust/Zig**. O GVL do MRI serializa CPU Ruby; concorrência é I/O-bound (ver `references/concorrencia.md`).
- **Correção/segurança de memória crítica** (auth, cripto, parsing de input hostil) → **Rust**.
- **Realtime/altíssima concorrência tolerante a falha** → **Elixir**.

**Framework — Rails vs. Sinatra/Roda (a escolha vira ADR):**
- **Rails** — aplicação com domínio rico, persistência relacional, jobs, admin. Traz ActiveRecord, migrations reversíveis, strong params, ActiveJob, generators. Preço: gravidade monolítica — contido por camadas (§4/§5) e `packwerk`.
- **Sinatra / Roda** — API leve / serviço fino / microserviço, sem o peso do Rails. Menos mágica, fronteira mais explícita, ideal quando não se precisa do full-stack.
- Servidor é **Puma**; jobs assíncronos em **Sidekiq** (ou ActiveJob sobre Sidekiq). Detalhe de concorrência em `references/concorrencia.md`.

> Regra de fit: um script/protótipo ou um Rails de produto pede Ruby; um hot path de throughput pede Go/Rust/Zig; auth/cripto pede Rust. Se dois encaixam, escolha o **default pragmático** (Go) e registre o porquê no ADR — ou registre por que Ruby vence aqui.

---

---

## 4. Arquitetura

**MUST — todos os projetos**
- Separação explícita de camadas: `domain`, `application`, `infrastructure`, `interface`.
- Inversão de dependência: domínio não conhece infra.
- **Domínio não importa Rails nem ActiveRecord.** POROs (Plain Old Ruby Objects) no domínio; a persistência ActiveRecord fica atrás de um **repository** em `infrastructure`. O `ActiveRecord::Base` é detalhe de infraestrutura, não entidade de negócio.

**SHOULD — projetos com regra de negócio relevante**
- DDD tático (agregados, value objects, eventos de domínio) — como POROs, não como models AR.
- Arquitetura hexagonal (ports & adapters). Model ActiveRecord fica **fino** (mapeamento/persistência), regra vive em service objects e entidades de domínio.

**MAY — CRUDs simples**
- Manter as 4 camadas, dispensar táticas DDD pesadas. Um Rails CRUD pequeno pode usar AR direto no use-case — desde que a fronteira exista e o controller não vire god-object.

### Dependências permitidas

```
interface       → application
application     → domain
infrastructure  → domain, application
```

### Dependências proibidas

```
domain          → qualquer outra camada
domain          → require 'active_record', 'sinatra', 'net/http', Sidekiq, ActionController
application     → interface
```

> No domínio, `require 'active_record'` ou referenciar `ApplicationRecord` é violação: o domínio não sabe que existe banco. Idem `require 'net/http'`/Faraday (rede) e `Sidekiq` (fila) — isso é infra.

### Anti-Corruption Layer

**MUST** em integrações com sistemas externos: adapter dedicado em `infrastructure/external/` que traduz o modelo externo para o modelo de domínio. **Nunca** expor o JSON cru / o response de gem de terceiro (Stripe, gateway, etc.) diretamente no domínio.

### 4.X DDD híbrido durante transição

Projetos Rails legados onde o `app/` já cresceu sem separação de camadas **podem** adotar DDD progressivamente em vez de big-bang. Regras:

**MUST**
- Toda nova feature/refactor em código tocado segue o layout completo (`app/domain/`, `app/application/`, `app/infrastructure/`, `app/interface/`) — não introduzir mais lógica de negócio dentro de fat model ou fat controller.
- Ao mover/quebrar arquivo legado, organize já em folders/packs DDD mesmo que internamente alguma classe ainda misture responsabilidades (ex.: service em `application/` ainda chamando `Model.where` direto). Estrutura primeiro, inversão depois.
- Cada PR que toca arquivo híbrido **deve** mover ao menos um pedaço pra direção certa (ex.: extrair regra de um fat model pra `domain/`, mover query pra um repository em `infrastructure/`).
- ADR registrando o débito e o plano de remoção: `<projeto>/<projeto>_archive/decisoes/<n>-ddd-migration-<contexto>.md`.

**SHOULD**
- Manter teste de cobertura por camada (ver a `schematize-qa`) durante a transição — domain começa com 0%, sobe a cada PR.
- Guard test ou `packwerk` que **rejeita dependências proibidas** logo que possível (mesmo com whitelist de exceções legadas):
  - `domain/` não referencia `ApplicationRecord`/`ActiveRecord`, `Sidekiq`, `ActionController`, `application/*`, `interface/*`.
  - `application/` não referencia `interface/*`.

**MAY**
- Marcar arquivos híbridos com comentário `# @ddd-hybrid` pra busca fácil e cleanup priorizado.

---

---

## 5. Estrutura de Pastas

Duas formas canônicas: **Rails com camadas DDD** e **serviço Ruby liso (Sinatra/Roda)**. Ambas mantêm a fronteira `domain → application → infrastructure → interface`.

### Rails (app com camadas DDD)

```
app/
├── domain/           # POROs: entities, value-objects, domain services, events, repository INTERFACES
├── application/      # use-cases / service objects, commands, queries, DTOs
├── infrastructure/   # repositories (impl AR), external/ (ACL de terceiros), messaging, persistence
├── interface/        # controllers, jobs (Sidekiq/ActiveJob), CLI/rake, serializers
├── models/           # ActiveRecord FINO — só mapeamento/persistência, sem regra de negócio
└── ...
lib/                  # utilitários de fronteira, tasks
config/               # rotas, initializers, credentials, database.yml
db/migrate/           # migrations reversíveis (change/up+down)
spec/                 # RSpec (ou test/ com Minitest), espelhando as camadas
packs/                # (opcional, com ADR) packwerk — fronteiras de módulo impostas
```

### Serviço Ruby liso (Sinatra / Roda)

```
lib/
├── domain/           # POROs, value-objects, repository interfaces
├── application/      # use-cases / service objects, commands, queries
├── infrastructure/   # repositories, external (ACL), persistence (Sequel/AR), messaging
├── interface/        # rotas Sinatra/Roda, jobs, CLI
├── shared/
└── config/
spec/
Gemfile / Gemfile.lock
.ruby-version
```

**Modular monolith em Rails — `packwerk` (packs):** quando o ADR de modular monolith (§2 MAY) for aprovado, a fronteira **não é convenção, é build**. Cada domínio vira um **pack** com `package.yml` declarando dependências permitidas e API pública (`public/`); violação de fronteira **falha o CI**. Sem `packwerk`, "módulo" é só uma pasta que todo mundo importa — big ball of mud com aparência de arquitetura.

> A regra de ouro em Rails: **o domínio não importa Rails**. Entidade de negócio é PORO; ActiveRecord é adapter de persistência atrás de repository. Se apagar o Rails o domínio ainda compila em `ruby` puro, a inversão está certa.

---

---

## 6. Complexidade e Tamanho

> **Canônico em `references/padroes-codigo.md`** (arquivo ≤ 750 linhas — ~500 de código útil + ~250 de comentário; flag em > 300 úteis; uma classe/módulo por arquivo; doc-comment YARD obrigatório com motivo/comportamento/entradas/saídas/efeitos; e `MAPA.md`). Esta seção é o recorte de arquitetura desses pisos — não duplica a regra, contextualiza.

**MUST — arquivos pequenos, métodos curtos**
- **Teto duro: ≤ 750 linhas/arquivo** (~250 de comentário + até ~500 de código útil). Acima disso, o arquivo **deve ser quebrado** — extraia responsabilidades em classes/módulos menores e a lógica em **métodos pequenos** com nome que explica a intenção. Não existe "model de 1200 linhas porque é coeso": fat model é o oposto de coeso.
- **Flag em > 300 linhas de código útil** (não bloqueia, mas **sempre sinaliza**): passou de 300 úteis (ou ~400 em observabilidade), é **indício** de classe/método muito extenso / falta de abstração — registre como dívida e **revise quando as prioridades forem resolvidas**.
- **Métodos pequenos e de responsabilidade única.** Ideal ≤ 15 linhas (norma Ruby/RuboCop `Metrics/MethodLength`); método grande vira métodos privados compostos ou service object. Use case: uma responsabilidade.
- Exceções (não disparam quebra): specs, migrations, código gerado, schemas/fixtures/factories, `schema.rb`.

**MUST — tudo documentado**
- **TODA classe/módulo público e método público carrega doc-comment YARD** (`# @param`, `# @return`, `# @raise`) declarando **o quê** + **onde é usado** (quem chama, em que fluxo/camada) + parâmetros, retorno e efeitos colaterais. Esse comentário é a **fonte do índice de microfunções** (§39) — escreva pensando que será extraído, não como enfeite.
- `# frozen_string_literal: true` no topo de **todo** arquivo Ruby.

**Bloqueio rígido em CI (RuboCop + gate)**
- Arquivo de produção > 750 linhas (ou > ~500 de código útil) sem quebra → bloqueia; > 300 úteis (~400 obs) → flag registrado.
- Classe/método público sem doc-comment YARD → bloqueia.
- Complexidade ciclomática > 15 (`Metrics/CyclomaticComplexity`/`PerceivedComplexity`) ou `Metrics/AbcSize` estourado → bloqueia.
- Aninhamento > 4 níveis (`Metrics/BlockNesting`).

> Linha de código é proxy ruim para complexidade — a ciclomática (e o ABC do RuboCop) é a métrica honesta. Mas fat model, método de 200 linhas e classe sem contexto são dívidas óbvias: quebre e documente antes do merge.

---

---

## 7. Dependências Internas e Shared Libraries

**MUST**
- Shared libraries (gems internas / engines) são **mínimas** e com escopo claramente delimitado.
- Permitido como shared: observabilidade, autenticação/auth, primitives de infraestrutura, SDKs internos, logging, configuração.

**MUST NOT**
- Criar gem/engine `commons` / `core-lib` / `platform-utils` genérico.
- Compartilhar **lógica de domínio** entre bounded contexts.
- Compartilhar entidades de domínio (nem models ActiveRecord). Cada contexto modela o seu.

> O caminho mais rápido pra um monólito distribuído é uma gem interna chamada `commons` — ou um Rails engine que todo serviço monta.

**SHOULD**
- Gems internas versionadas com SemVer próprio; pin exato no `Gemfile.lock` (lockfile é piso, §cadeia-suprimentos).
- Breaking changes em gem interna exigem ADR.

---

---

## 8. CQRS e Padrões de Aplicação

- **Commands**: alteram estado, retornam id ou void. Modelados como service objects (`Application::CreateOrder.call(...)`).
- **Queries**: nunca alteram estado, otimizadas para leitura, podem usar projeções / scopes AR read-only.
- CQRS **não exige** event sourcing.

---

---
