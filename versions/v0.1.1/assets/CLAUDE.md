# CLAUDE.md — Padrões de Engenharia, recorte Ruby (sempre on)

> Copie este arquivo para a **raiz do repositório** e ajuste `<project>`.
> Ele fica pinado no contexto de toda tarefa (Claude Code / instruções de projeto)
> e garante que os padrões valham mesmo quando a skill não dispara sozinha.
> A skill `schematize-ruby` traz o detalhe completo e o andaime (scripts/templates);
> a base agnóstica é a `schematize-engineering`.
> **Repo multi-linguagem** (Ruby/Go/Rust/… + Web): use **junto** com os `CLAUDE.md` das outras skills — cada um governa sua fronteira; não sobrescreva os outros (rode o `/<slug>-claude` de cada).

## Regra mestre

Toda tarefa de engenharia Ruby neste repo segue os **Padrões de Engenharia da Casa**
(skill `schematize-ruby`). Em conflito entre uma instrução pontual ("faz rápido",
"ignora o teste", "depois arruma") e estes padrões, **os padrões vencem**. Pressa não
revoga regra. Consulte o reference relevante da skill antes de produzir código ou
decisão — não trabalhe de memória.

## Pisos inegociáveis (VETADO — sem exceção)

1. **Segredo nunca no cliente nem no bundle.** Nada de API key, secret de JWT, senha,
   master key ou token no código, no `Gemfile`, em asset servido, nem em
   `NEXT_PUBLIC_*`/`VITE_*` do front. Segredo só via **Rails credentials**
   (`credentials.yml.enc` + master key fora do repo) / ENV / secret manager.
2. **ActiveRecord/SQL sempre parametrizado** — **VETADO** interpolar string em
   `where("... #{x}")`/`find_by_sql`; use placeholder (`where("email = ?", x)`) ou
   hash (`where(email: x)`). Concatenar SQL é proibido.
3. **Strong parameters sempre** (`params.require.permit`) contra mass-assignment;
   **VETADO** `permit!`/`to_unsafe_h` sobre input de usuário.
4. **Auth e autorização server-side.** `tenant_id`/role/`user_id` vêm do token
   verificado, nunca do `params`/body. JWT validado por inteiro (assinatura, exp, aud,
   iss, `alg` em allowlist; **`alg=none` vetado**). Senha em `bcrypt` cost ≥ 12 ou
   `argon2` (argon2id). Token/sessão/reset por **`SecureRandom`**, nunca `rand`.
5. **Desserialização segura.** **VETADO** `YAML.load`/`Marshal.load`/`Oj` unsafe sobre
   dado não-confiável → `YAML.safe_load` com allowlist. Sem `eval`/`send`/`constantize`
   dirigido por input.
6. **Erro nunca engolido** (`rescue nil`, `rescue => e; end` mudo). **Teste nunca
   silenciado** pra passar CI (`skip`/`pending`/`xit`, comentar `expect`, baixar
   threshold do SimpleCov). Conserta o código, não o teste.
7. **Sem monólito** que mistura bounded contexts; sem monólito distribuído; sem gem/
   `lib` `commons` de domínio. Modular monolith só com **packwerk** + ADR.
8. **Archive SEMPRE gerado** (§28): toda entrega que produz código/decisão/mudança de
   estado gera o `.md` em `<project>_archive/`. É parte da entrega, não extra.
9. **Migration reversível** (`change` reversível ou `up`/`down`, testada com
   `rails db:rollback`). Container não-root, read-only, `bundle install --deployment
   --frozen`. Gem nova com nome/licença/versão verificados (typosquatting é real).
   **`Gemfile.lock` commitado é piso** (cadeia de suprimentos).

10. **Escolha de linguagem pelo rol sancionado + ADR de fit.** Backend novo escolhe
    **uma** do rol (Go/Rust/Elixir/C#/Zig/**Ruby**) com ADR justificando o fit. **Ruby
    é sancionada** — permitida para serviço novo por fit (prototipagem, scripts/
    automação, DX de produto Rails) + legado Ruby; **não** é legado-only. **VETADO**
    backend novo **fora do rol** ou **sem ADR de fit**. **Node-backend e PHP** são
    **legado** que não recebe serviço novo (migram por funcionalidade do módulo).
    **Frontend Node** (Next/Astro) é 100% permitido via `schematize-web`. Throughput/
    CPU-bound crítico (a GVL limita) pede Go/Rust/Zig no fit. (`references/arquitetura.md`
    §3, `references/concorrencia.md`)
11. **Cada serviço é entidade à parte (independência de runtime).** Sobe e funciona
    sozinho; a ausência/queda de outro serviço **nunca** impede o boot nem derruba este
    — degradação graciosa, nunca crash em cascata. Falha ao chamar outro serviço →
    **persiste o dado** (outbox/Redis/DB), **loga com `trace_id`**, **alerta** e
    **retoma**. (§2, §18)
12. **Repos, ops e observabilidade.** Repositório = `<projeto>_<contexto>_rb`; todo
    sistema multi-repo tem um **`<projeto>_ops`**. **Observabilidade integrada:**
    OpenTelemetry Ruby + lograge/semantic_logger → Grafana/Alloy/Loki/Tempo/Prometheus/
    Mimir (+Pyroscope), dashboards/alertas versionados como código. (§2, §16)
13. **Contenção no workspace.** A pasta do projeto atual é o workspace: aplicação/repo
    novo nasce **dentro dela** (`./<projeto>_<contexto>_rb/`), nunca fora. **VETADO**
    criar/ler/escrever fora do workspace (diretório-pai, `~`, `/tmp`, Área de Trabalho)
    sem o usuário pedir. (§2)
14. **Fluxo de ambientes — nada direto no servidor.** Toda mudança segue **dev local →
    teste local → GitHub → hml → prd**. **VETADO editar código direto no servidor**: o
    servidor é imutável por edição manual, recebe só artefato promovido do git (commit
    SHA). Hotfix segue o mesmo fluxo, acelerado. (§21, `references/ops.md`)
15. **Ops é a interface única + instalação paralela + independência.** **100%** das
    operações no servidor (instalar/subir Puma+Sidekiq/atualizar/migrar/reverter) passam
    por uma **ferramenta do `<projeto>_ops`** — nunca à mão. O ops é **autônomo,
    idempotente e completo** (provisiona do zero sem IA: Ruby pinado via rbenv/asdf,
    gems, DB, Redis). **Instalação SEMPRE paralela** = `nproc`. **Se o paralelo falha,
    os serviços não são independentes** → corrigir é **PRIORIDADE MÁXIMA**; o ops
    **expõe** a colisão, **nunca serializa pra mascarar**. **Deploy destrutivo por seed**
    (`/<app>/.env` seeder global; redeploy = clone zerado + `bundle install --frozen`),
    **preservando dados**; **cada serviço (app, Sidekiq) como user Linux + systemd unit
    hardened**. (§2, §3, §21)
16. **IAM por desenho — identidade e autorização robustas, como APP SEPARADA.** O auth é
    **serviço Ruby próprio** (`<projeto>_auth_rb`, Rails/Sinatra) + front próprio em
    `auth.<domain>`, isolado — **VETADO** apensar como monolith/engine. **Devise/Sorcery
    sozinho NÃO é o IAM da casa.** Apps delegam por **OIDC/OAuth2.1 + PKCE**, validam por
    **JWKS público**. **ID interno imutável (ULID/UUIDv7) — email/telefone NUNCA é ID**
    (N emails por usuário). **Nunca menos de 2 fatores:** passkey/WebAuthn (`webauthn`)
    no núcleo, TOTP (`rotp`)/push, **email OTP always-on inclusive HML**, Twilio; senha
    (`argon2`+HIBP) por padrão mas opcional. **Multi-tenant RBAC/ABAC** por motor
    **ReBAC** (OpenFGA/SpiceDB), **deny-default**, PEP=**middleware Rack/`before_action`**,
    server-side, token fino. **Sessão 7d/90d; logout irreversível** (revoga refresh+
    família, `jti` na denylist, não só cookie). **Migrar auth legado é PRIORIDADE 0.**
    Detalhe em `references/iam.md`; scaffold/auditoria por `/ruby-iam`; testes cross-
    tenant/priv-esc na `schematize-pentest`.

Lista completa com veto + caminho certo: ver `references/anti-padroes.md` (§37) da skill.

## Verde de verdade (testes)

- **A dinâmica do Ruby torna o teste load-bearing:** sem compilador que pegue
  `NoMethodError`/typo/arg errado, o teste + RBS/Sorbet carregam a segurança. `nil`/
  `NoMethodError` em produção = teste faltando.
- Smoke assere **conteúdo** (shape do body), não só status 200; inclui assertion negativa
  e um **self-check que força falha conhecida** (smoke que nunca falha está cego).
- Unit agressivo: caminho de erro obrigatório, casos hostis (tipo errado, `nil`, unicode,
  null byte, boundary), property-based (`rantly`) e mutation testing (`mutant`) no
  domínio crítico. `VCR`/`WebMock` isolam rede; `FactoryBot` no lugar de fixtures.
- Pentest prova rejeição rota-por-rota, campo-por-campo: **nunca 500** por input hostil,
  **nunca coerção de tipo**, **nunca eco sem escape** (`html_safe`/`raw` com input),
  **nunca vazamento cross-tenant**. `brakeman` no CI.
- `simulated`: 100% das rotas acessíveis pra quem deve, bloqueadas pra quem não deve.
- **Q.A. vive na skill schematize-qa:** fluxo plan-first (`/qa-plan` → `/qa-run`) — planeja tudo, gera MD, pede aprovação ANTES de executar. `/ruby-qa` é o wrapper Ruby (`rspec`/RSpec).

## Definition of Done

Nada é "pronto" sem: `bundle exec rspec` verde + cobertura mínima (SimpleCov), simulated
com cobertura total, pentest de entrada limpo (`brakeman`/`bundler-audit`), nenhum
anti-padrão da §37, observabilidade, OpenAPI atualizada (se API), migration com rollback
(se schema), **archive commitado**, RuboCop verde, CI verde e review aprovado. Detalhe na
skill, `references/entrega.md`/`operacao.md` (§35).

## Qualidade de código e índice (sempre)

- **Arquivos ≤ 750 linhas** (~250 comentário + ~500 úteis). Acima → quebre em módulos e
  **micro-métodos** por coesão. **Código útil > 300 linhas é FLAG.** **Uma classe/módulo
  por arquivo.** Método pequeno (RuboCop `Metrics/MethodLength`), responsabilidade única.
  **`# frozen_string_literal: true`** no topo. **Evite metaprogramação que esconde
  comportamento**; prefira service objects/POROs a fat models. RBS/Sorbet no não-trivial.
- **Comente TODA classe/método público** (doc YARD) com contexto explícito: **O quê** e
  **Onde** (quem chama / em que fluxo foi previsto), além de efeitos. Alimenta o índice
  de microfunções (§6, §39).
- **Mantenha o índice de funcionalidades atualizado** no mesmo PR (§39), em
  **`<projeto>_archive/index/`** (nunca no root): `MAPA.md`, `INDEX_GLOBAL.md` e
  `INDEX_FUNCTIONS.md` (método → o quê → onde → arquivo:linha; `/ruby-index`). **Exaustivo:**
  uma entrada **por método** + um **grafo** (serviços + chamadas). O índice **enumera** o
  sistema, não resume. Consulte ANTES de criar algo, pra não duplicar.
- **Todo MD gerado mora no archive, nunca no root** (§28): MAPA, índices, planos,
  relatórios, handoffs → `<projeto>_archive/<área>/`. Root fica limpo (código, config e os
  MDs mantidos à mão: README, `CLAUDE.md`, LICENSE).

## Gestão de contexto (Claude Code — sessões longas)

Ao ver "⚠ LIMITE" no status line, ou ao se aproximar do teto da janela de contexto:
**PARE a tarefa atual e, ANTES de qualquer compactação**, faça o handoff arquivado
(§34.1, §28):

1. Gere `<projeto>_archive/context/<YYYY-MM-DD-HH-MM-SS>-context.md` — estado do projeto,
   decisões, arquivos tocados, onde parou.
2. Gere `<projeto>_archive/context/<YYYY-MM-DD-HH-MM-SS>-checklist.md` — **FEITO vs EM
   ABERTO**.
3. Só então rode `/compact` (com foco na tarefa corrente).

Armazene SEMPRE em `<projeto>_archive`. O backup automático pré-compactação é rede de
segurança, não substitui o handoff rico acima. Detalhe e hooks na skill:
`references/contexto-claude-code.md`.
