---
name: schematize-ruby
metadata:
  version: 0.2.0
description: Padrões normativos de engenharia da casa no recorte Ruby (Rails 8, Sidekiq, RSpec) — arquitetura/DDD, segurança, IAM, testes/pentest, dados, observabilidade, deploy, archive. Use SEMPRE que for projetar, gerar, revisar ou refatorar backend, API, serviço, job, schema, migration, infra, CI/CD, teste ou deploy em Ruby/Rails — mesmo sem citar "padrão" —, e ao escolher stack (Ruby entra por fit + ADR no rol Go/Rust/Elixir/C#/Zig/Ruby), modelar eventos/banco, desenhar auth, configurar observabilidade ou produzir ADR/runbook/archive. Pisos: segredo nunca no cliente; nada de SQL por interpolação (AR/Sequel parametrizado); auth server-side com JWT validado por inteiro; IAM como app separada; efeito externo (e-mail/SMS/push) NUNCA sai de não-produção — sink por default, guard deny-by-default, cap por execução, domínio de teste em rota nula; archive obrigatório. Frontend delega à schematize-web; segurança ofensiva à schematize-pentest; disciplina de teste à schematize-qa.
---

# Padrões de Engenharia da Casa — recorte Ruby

Conjunto normativo que rege como software Ruby é projetado, construído, testado e operado aqui. Especializa a base agnóstica (`schematize-engineering`) para o idioma, o build (Bundler/gems), os frameworks (Rails/Sinatra/Roda/Sidekiq/Puma), o modelo de concorrência (GVL/threads/Ractor/async) e o ecossistema Ruby — **sem nunca afrouxar o piso**. Escolher Ruby é decisão de ADR (fit); cumprir o piso não é opcional.

**Versão:** skill `schematize-ruby` v0.2.0. Changelog em `CHANGELOG.md`.

## Precedência e herança (leia antes de divergir)

Esta skill é o **recorte Ruby** da base. Duas regras governam a relação, e elas resolvem sozinhas
quase toda dúvida de "onde está escrito o quê":

1. **Onde esta skill divergir da base, a BASE MANDA.** `schematize-engineering` é a normativa; aqui
   mora a **especialização** — o mecanismo, a lib, a sintaxe, o gate da linguagem. Divergência de
   *piso* entre este arquivo e a base é **defeito desta skill**, não uma variante local aceitável.
   Achou uma? É item de correção, não licença. *(Foi assim que o `argon2id-only` da casa virou
   "argon2id ou PBKDF2" em uma skill só, e o rol de 6 linguagens virou "só Go e Rust" em três.)*
2. **O que não está repetido aqui é HERDADO, não dispensado.** A ausência de um piso neste repo
   nunca significa que ele não vale — significa que ele não muda de forma nesta linguagem. Em
   especial, valem integralmente, sem cópia local:
   - **§28 Archive** — `<projeto>/<projeto>_archive/` é **repositório git próprio, PRIVADO e
     obrigatório**, criticidade 0 (`schematize-archive`; ADR-0005 para a planta canônica).
   - **§39 Índice/MAPA** — enumeração exaustiva (uma entrada por unidade chamável, `M == N`) e o
     **grafo com arestas em ASCII (`A -> B`), NUNCA a seta unicode** — o parser do app lê ASCII.
   - **§35 Definition of Done** e a lista de anti-padrões **§37** (citada por **título**, nunca por
     número: a numeração dos itens diverge entre skills).
   - **IAM** (`schematize-engineering` → `references/iam.md`): identidade ≠ email, ≥2 fatores, ReBAC multi-tenant,
     **alcançabilidade do 2º fator** (o fator de recuperação tem de ser alcançável quando o
     principal cai — senão o 2FA vira bug de bootstrap que tranca o dono para fora), os parâmetros
     mínimos de argon2id, sessão longa e logout irreversível.
   - **Rol sancionado** — Go, Rust, Elixir, C#, Zig, Ruby, por **fit + ADR**
     (`schematize-engineering` → `references/linguagens.md`). Esta skill é **uma** delas, não a
     régua das outras.
   - **Efeito externo** nunca sai de não-produção (`schematize-engineering` →
     `references/efeitos-externos.md`; gate em `scripts/check-external-effects.sh`, distribuído
     aqui — ADR-0008).

## Comandos (Claude Code)

Digite `/ruby-help` pra ver todos. Em resumo:

| Comando | O que faz |
|---|---|
| `/ruby-help` | lista todos os comandos do schematize-ruby |
| `/ruby-load` | carrega à força todo o corpo normativo no contexto e passa a aplicá-lo |
| `/ruby-claude` | cria/atualiza (sobrescreve) o `CLAUDE.md` da raiz com a versão atual da skill |
| `/ruby-cc` | context compact: gera context.md + checklist.md no archive e roda `/compact` |
| `/ruby-handoff` | gera o handoff (context.md + checklist.md) **sem** compactar — pra fim de sessão |
| `/ruby-qa` | Q.A. no contexto Ruby — wrapper fino da **schematize-qa** (`/qa-plan` → `/qa-run`) com `rspec`/RSpec |
| `/ruby-review` | roda o gate da DoD/§37 no diff (arquivo >750 bloqueia / >300 úteis flag, método sem doc YARD, SQL interpolado, `rand` pra token, índice) |
| `/ruby-iam` | força/audita/scaffolda o IAM (identidade≠email, ≥2 fatores, ReBAC multi-tenant, sessão longa/logout irreversível) como **app Ruby separada** em `auth.<domain>`, ou porta um auth legado |
| `/ruby-index` | (re)gera o índice de microfunções (§39) a partir dos doc-comments YARD |
| `/ruby-ops` | audita/scaffolda o `<projeto>_ops` (interface única): fluxo de ambientes, instalação paralela (`nproc`), independência |

Os comandos ficam em `assets/commands/` e são instalados em `.claude/commands/`.

## Como usar esta skill

1. Identifique o domínio da tarefa e **leia o(s) reference(s) relevante(s)** antes de produzir código ou decisão. Não trabalhe de memória — os detalhes (versões, limites, convenções, gems) estão nos arquivos.
2. **Sempre** aplique os pisos inegociáveis abaixo, independente do reference carregado.
3. Ao terminar, valide contra a Definition of Done (`references/entrega.md`, §35) e **gere o archive** (§28, `references/operacao.md`).

Mapa de references — leia o que casa com a tarefa:

| Tarefa | Reference |
|---|---|
| **Limites de código (arquivo ≤750: ~500 úteis + ~250 comentário; flag >300 úteis), uma classe/módulo por arquivo, doc-comment YARD, `frozen_string_literal`, RuboCop, RBS/Sorbet, MAPA** | `references/padroes-codigo.md` |
| **Escolha de linguagem (rol sancionado + fit de Ruby, §3)**, arquitetura, camadas, DDD, packwerk, repository pattern isolando ActiveRecord, estrutura de pastas Rails/Sinatra | `references/arquitetura.md` |
| Eventos/mensageria (Sidekiq/ActiveJob), banco (ActiveRecord/Sequel), migrations reversíveis, cache Redis, APIs, resiliência, outbox | `references/dados-eventos.md` |
| Segurança, auth, JWT, multi-tenancy, LGPD, **segredos/ActiveRecord parametrizado/strong params/SecureRandom/YAML seguro/brakeman** | `references/seguranca.md` |
| **IAM (identidade+autorização): auth como app Ruby separada (`auth.<domain>`), Devise sozinho não é o IAM, ID≠email, ≥2 fatores (`webauthn`/`rotp`/OTP), ReBAC multi-tenant, sessão longa/logout irreversível, migração de legado** | `references/iam.md` |
| **Cadeia de suprimentos: `Gemfile.lock`, `bundler-audit`, SBOM, imagem mínima/pinada, `.ruby-version` não-EOL** | `references/cadeia-suprimentos.md` |
| **Concorrência: a GVL, threads (I/O), Puma/forking e Ractor (CPU), async/fibers, Sidekiq** | `references/concorrencia.md` |
| Testes — o recorte Ruby (runner, sintaxe, armadilhas do dialeto). **A disciplina é da `schematize-qa`.** | `references/testes.md` |
| Observabilidade (OTel Ruby, lograge/semantic_logger), healthchecks, performance, FinOps | `references/observabilidade.md` |
| Config, deploy, git/PR, ownership, runbooks/incidentes, ADR, **archive** (§20–28) | `references/operacao.md` |
| **Ops (control plane): fluxo dev→local→github→hml→prd, ops interface única, instalação paralela=`nproc`, independência=invariante** | `references/ops.md` |
| Templates, feature flags, IA assistida, DoD, evolução, índice de funcionalidades (§29+) | `references/entrega.md` |
| **Versões e EOL de Ruby/Rails + limiares de gems** (fonte volátil) | `references/stack-versoes.md` |
| Filosofia, aplicação universal e a lista completa de anti-padrões vetados | `references/anti-padroes.md` |
| Gestão de contexto em sessões longas no Claude Code (handoff, hooks) | `references/contexto-claude-code.md` |

## Pisos inegociáveis (VETADO — sem ADR de exceção)

Estes nunca são violados, nem "pra funcionar", nem "pra ir mais rápido". A lista completa com veto + caminho certo está em `references/anti-padroes.md` (§37). Os que mais aparecem em Ruby gerado às pressas:

- **Segredo nunca no cliente nem no bundle.** Nada de API key, secret de JWT, senha de banco ou token no código, no `Gemfile`, em asset servido ou em `NEXT_PUBLIC_*`/`VITE_*` do front. Segredo só via **Rails credentials** (`credentials.yml.enc` + master key fora do repo) / ENV / secret manager. Detalhe em `references/seguranca.md`.
- **ActiveRecord/SQL sempre parametrizado.** **VETADO** interpolar string em `where("... #{x}")`, `find_by_sql`, `sanitize_sql` mal usado — use placeholders (`where("email = ?", x)`) ou hash (`where(email: x)`). Concatenar SQL é injeção esperando acontecer.
- **Strong parameters sempre.** `params.require(...).permit(...)` contra **mass-assignment**; **VETADO** `permit!`/`params.to_unsafe_h` sobre input de usuário.
- **Auth e autorização server-side.** `tenant_id`/role/`user_id` vêm do token verificado, nunca do `params`/body. Validação no front é UX, não controle. JWT validado por inteiro (assinatura, exp, aud, iss, `alg` em allowlist — **VETADO `alg=none`**). Senha em `bcrypt` cost ≥ 12 ou `argon2` (argon2id) + HIBP. Token/sessão/reset por **`SecureRandom`**, nunca `rand`/`Random`.
- **Desserialização segura.** **VETADO** `YAML.load`/`Marshal.load`/`Oj` unsafe sobre dado não-confiável → `YAML.safe_load` com allowlist. Sem `eval`/`send`/`constantize` dirigido por input.
- **Erro nunca engolido** (`rescue nil`, `rescue => e; end`, `rescue StandardError; end` mudo). Sem calar o problema pra "seguir o fluxo".
- **Teste nunca silenciado** pra passar CI (`skip`/`pending`/`xit`, comentar `expect`, baixar threshold do SimpleCov). Conserta o código, não o teste. **A dinâmica do Ruby torna o teste load-bearing:** sem compilador que pegue `NoMethodError`/typo/arg errado, o teste + RBS/Sorbet carregam a segurança.
- **Sem monólito que mistura bounded contexts**, sem monólito distribuído, sem gem/`lib` `commons` de domínio compartilhada. Fronteira de módulo por **packwerk** quando modular monolith (com ADR). Detalhe em `references/arquitetura.md`.
- **Archive SEMPRE gerado.** Toda entrega que produz código/decisão/mudança de estado gera o `.md` de archive (§28) — parte da entrega, não extra. Templates em `assets/`.
- **Migration reversível** (`change` reversível ou `up`/`down`, testada com `rails db:rollback`). Container não-root, read-only, `bundle install --deployment --frozen`. Gem nova com nome/licença/versão verificados (typosquatting no rubygems.org é real).
- **Efeito externo NUNCA sai de não-produção.** E-mail, SMS/voz, push, webhook de terceiro e cobrança **não acontecem de verdade** fora de `production` — por construção. Em Rails: `config.action_mailer.delivery_method` é **`:test`** no test, **`:letter_opener`/Mailpit SMTP** no dev e no staging, **real só em `production`** (com `ENV.fetch` sem default — sem credencial o boot **falha**, não sinka em silêncio); e o guard é um **INTERCEPTOR** (`ActionMailer::Base.register_interceptor`), porque interceptor pega **TODO mailer, inclusive os que alguém escrever depois** — destinatário fora do domínio de teste com `Rails.env != "production"` **levanta erro** e aborta a entrega, nunca warning. **Cap por execução** (`MAIL_MAX_PER_RUN`, default 50) com abort, **fail-closed** (só a string exata `"production"` entrega de verdade; o interceptor é registrado **sem condição de ambiente**), **chave sandbox** fora de prd. Todo endereço sintético (spec, fixture, FactoryBot `sequence`, `db/seeds.rb`, rake) é `<papel>+<run-id>-<n>@test.<domain>` — domínio em **rota nula** (null MX RFC 7505 + SPF `-all` + DMARC `p=reject`) ou TLD reservado (`.test`/`.invalid`/`.example`); **VETADO** `@gmail.com`, domínio de terceiro, e-mail de pessoa real (inclusive o seu) e o domínio de produção. Entregar de verdade fora de prd exige **as cinco** (ADR + allowlist ≤5 + cap + janela + subdomínio separado). Motivo: bounce/complaint em massa **queima IP e domínio** e derruba o transacional de prd — inclusive o **OTP de login**. Código, interceptor e specs em `references/iam.md` §3.1; normativa em `schematize-engineering` → `references/efeitos-externos.md`.
- **Pisos de código (`references/padroes-codigo.md`):** arquivos **≤ 750 linhas** (~250 comentário + ~500 úteis; acima → quebrar por coesão), **flag em > 300 úteis**, **uma classe/módulo por arquivo**, **método pequeno** (RuboCop `Metrics/MethodLength`), **`# frozen_string_literal: true`** no topo, **toda classe/método público com doc-comment YARD** (O quê / Onde / entradas / saídas / efeitos), **`MAPA.md`** atualizado no mesmo PR — em **`<projeto>_archive/index/`, nunca no root** — e **índice de microfunções** regenerado (`/ruby-index`). **Evitar metaprogramação que esconde comportamento.** Todo MD gerado (MAPA/índice/plano/relatório/handoff) mora no archive, root limpo (§28).
- **Escolha de linguagem pelo rol sancionado + ADR de fit (`references/arquitetura.md` §3).** Backend novo escolhe **uma** do rol (Go/Rust/Elixir/C#/Zig/**Ruby**) com ADR justificando o fit. **Ruby é sancionada** — permitida para serviço novo por fit (prototipagem, scripts/automação, DX de produto Rails) + manutenção de legado Ruby; **não** é legado-only. **VETADO** backend novo **fora do rol** ou **sem ADR de fit**. Node-backend e PHP são **legado** que não recebe serviço novo (migram por funcionalidade do módulo). Frontend Node (Next/Astro) é 100% permitido via `schematize-web`. Quando throughput/CPU-bound é crítico (a GVL limita — `references/concorrencia.md`), o fit pede Go/Rust/Zig.
- **Fluxo de ambientes e ops (`references/ops.md`).** Toda mudança segue **dev local → teste local → GitHub → hml → prd**; **VETADO editar código direto no servidor**. **100%** das operações no servidor passam pela **ferramenta do `<projeto>_ops`** — nunca à mão; o ops é **autônomo** (usuário provisiona do zero sem IA). **Instalação sempre paralela** = `nproc`; **falha no paralelo = serviços não independentes → corrigir a independência é prioridade máxima** (não serializar pra mascarar).
- **Deploy destrutivo por seed + isolamento por usuário (`references/ops.md` §2–§3).** O ops provisiona em **`/<app>/`** clonando os repos dentro; **`/<app>/.env` é o seeder global**. **Todo redeploy é destrutivo na aplicação** (clone zerado do seed, `bundle install --frozen`), **preservando os dados** (migration reversível). **Cada serviço (app Puma, Sidekiq) roda como user Linux próprio em systemd unit hardened.**
- **IAM por desenho (`references/iam.md`).** Todo projeto começa com identidade+autorização robustas, e o **auth é app SEPARADA** — **serviço Ruby próprio** `<projeto>_auth_rb` + front em `auth.<domain>`, isolados; nunca apensado como monolith/engine. **Devise/Sorcery sozinho não é o IAM da casa.** Apps delegam por OIDC/PKCE e validam por JWKS público. **ID interno imutável (ULID/UUIDv7) — email/telefone nunca é ID.** **≥2 fatores sempre** (passkey `webauthn` no núcleo, TOTP `rotp`/push, email OTP always-on, Twilio; senha `argon2`+HIBP por padrão mas opcional). **Multi-tenant RBAC/ABAC via ReBAC** (OpenFGA/SpiceDB; deny-default, PEP=middleware Rack/`before_action`, server-side, token fino). **Sessão 7d/90d, logout irreversível.** **Migrar auth legado = prioridade 0.** Scaffold/auditoria por **`/ruby-iam`**; testes cross-tenant/priv-esc na `schematize-pentest`.

> Regra de bolso: se a justificativa começa com "só pra funcionar", "depois eu arrumo" ou "é mais rápido assim" e o resultado mexe em segredo, auth, dado ou registro — é um anti-padrão vetado. Pare e faça certo.

## Testes — o que conta como "verde de verdade"

Detalhe em `references/testes.md` (o recorte Ruby) — a **disciplina** de teste é da `schematize-qa`, e a segurança ofensiva da `schematize-pentest`.

- **Smoke não pode ser teatro:** assertar shape do body (não só status 200), assertion negativa (sem stack trace/placeholder), e um **self-check que força uma falha conhecida** pra provar que o runner reporta FAIL. Smoke que nunca falha está cego.
- **Unit agressivo:** caminho de erro obrigatório, casos hostis (tipo errado, `nil`, unicode, null byte, boundary), property-based (`rantly`) e mutation testing (`mutant`) no domínio crítico. **`NoMethodError`/`nil` em produção = teste faltando.** Cobertura (SimpleCov) é piso, não meta.
- **Fronteiras isoladas:** `VCR`/`WebMock` bloqueiam rede real; `FactoryBot` no lugar de fixtures gigantes.
- **Pentest prova rejeição, rota por rota, campo por campo:** nunca 500 por input hostil, nunca coerção silenciosa de tipo, nunca eco sem escape, nunca vazamento cross-tenant. `brakeman` no CI. Princípios em a `schematize-pentest`).
- **`simulated` (teste emulado):** cruza rotas × personas × injections e prova que **100% das rotas** estão acessíveis pra quem deve e bloqueadas pra quem não deve. Rota fantasma/morta quebra o run.
- **Q.A. vive na skill dedicada schematize-qa:** o fluxo plan-first (`/qa-plan` → `/qa-run`) planeja tudo, gera o MD, **pede aprovação antes de executar** e trava nos gates. `/ruby-qa` é o wrapper do recorte Ruby (`rspec`/RSpec). Nada de Q.A. às cegas.

## Andaime pronto (scripts e templates)

Não escreva do zero o que já está bundlado:

- `scripts/lib.sh` — helpers de teste (`test_pass`, `test_fail`, `test_skip`, `test_section`, `test_summary`, `http_call`, `assert_http_in`). Todo script de teste usa estes.
- `scripts/test-skeleton.sh` — esqueleto de `tests/<mode>/<name>.sh`.
- `scripts/smoke-selfcheck.sh` — o meta-teste anti "verde mentiroso".
- `scripts/simulated/run.py` — scaffold do engine rotas × personas × injections.
- `scripts/hooks/context-monitor.mjs` + `scripts/hooks/precompact-backup.mjs` — gestão de contexto no Claude Code (backup automático em `<projeto>_archive` antes da compactação). Ver `references/contexto-claude-code.md` e `assets/settings.claude.example.json`.
- `assets/ADR.md`, `assets/TASK.md`, `assets/CHAT_ARCHIVE.md`, `assets/PR_TEMPLATE.md`, `assets/RUNBOOK.md` — templates (cumprem §27/§28).
- `assets/INDEX_GLOBAL.md` + `assets/INDEX_FUNCTIONS.md` — índice de funcionalidades (§39): o global é mantido à mão; o de microfunções é **gerado** dos doc-comments YARD (§6) via `/ruby-index`.
- `assets/lint/` — configs de guard/lint (adapte para `.rubocop.yml` com os `Metrics/*` mapeando os limites de código).
- `assets/CLAUDE.md` — arquivo "sempre on" pra colocar na **raiz do repositório**: pina estes padrões no contexto de toda tarefa. Copie e ajuste `<project>`.
- `assets/commands/ruby-cc.md` — comando `/ruby-cc` (context compact). Copie para `.claude/commands/ruby-cc.md`.

## Aplicação sempre-on

Esta skill é puxada quando a tarefa casa com a descrição. Para garantir que os padrões valham em **toda** interação do repo (e não só nas que disparam a skill), copie `assets/CLAUDE.md` para a raiz do projeto. Os dois mecanismos se complementam: o `CLAUDE.md` pina o resumo e aponta pra cá; a skill entrega o detalhe e o andaime. Em repo com várias linguagens do rol, use junto com os `CLAUDE.md` das skills irmãs (cada um governa sua fronteira) e com a base `schematize-engineering`.
