# Changelog — schematize-ruby

Todas as mudanças relevantes deste pacote, no formato [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/),
com versionamento [SemVer](https://semver.org/lang/pt-BR/).

## [0.3.1] — 2026-08-21

### Corrigido
- **`padroes-codigo.md`: o exemplo de endless method não compilava.** `def call(params) = ...` é **SyntaxError** (✔ verificado no Ruby 3.4, em container: *unexpected end-of-input; expected an expression after the operator*) — o `...` em Ruby é **argument forwarding**, não reticência de exemplo. Corrigido para um corpo real, com a nota que explica a diferença: *exemplo que não compila é copiado assim mesmo, e o leitor perde tempo achando que errou o resto*.

## [0.3.0] — 2026-08-21
Saneamento do catálogo conforme a vistoria de 2026-08-21.

### Mudado
- **Solid Queue é o default** de jobs (`concorrencia.md` §C6), com o motivo que é de arquitetura e não de gosto: com Sidekiq a fila vive num **Redis fora da transação** do dado, e é dali que nasce o job que roda **antes do commit** e processa um registro que não existe; com Solid Queue o enfileiramento é uma linha no mesmo Postgres — **o outbox de graça**. Sidekiq continua legítimo, com ADR, quando o volume/latência pede.
- Entrou o MUST que faltava e é o bug de fila mais comum do Rails: **enfileirar depois do commit**.

### Corrigido
- **`authenticate_by` (Rails 7.1+) prescrito** em `seguranca.md` e `iam.md`, com o mecanismo: escrito ao pé da letra, `find_by(email:)&.authenticate(...)` **não executa o hash quando o e-mail não existe** e responde muito mais rápido — **enumeração de usuário por timing**, medível de fora. O par obrigatório é a resposta genérica idêntica nos dois casos.
- Hash de senha alinhado ao piso da casa: **argon2id para senha nova**, bcrypt cost ≥ 12 **só como legado a migrar** (era "bcrypt **ou** argon2id" — a deriva da Classe C).

### Mudado (cont.)
- `anti-padroes.md`, `arquitetura.md`, `entrega.md`, `dados-eventos.md`, `observabilidade.md`, `seguranca.md`, `cadeia-suprimentos.md` e `ops.md` viraram **ponteiro** (poda mecânica).

## [0.2.0] — 2026-08-20
Piso novo: **efeito externo NUNCA sai de não-produção** — recorte Ruby/Rails (ActionMailer + interceptor) da normativa `schematize-engineering` → `references/efeitos-externos.md`.
### Adicionado
- **SKILL.md**: piso inegociável "efeito externo NUNCA sai de não-produção" — `delivery_method` sinkado fora de `production`, guard como **interceptor** (pega todo mailer, inclusive os futuros), cap por execução, fail-closed, domínio de teste em rota nula.
- **references/iam.md §3.1** (novo): três camadas com código Ruby real — (1) `config/environments/*.rb` com `delivery_method` `:test` / `:letter_opener`+Mailpit / Mailpit no staging / SMTP real só em production com `ENV.fetch` sem default; (2) `lib/external_recipient_guard.rb` — `ExternalRecipientGuard#delivering_email` deny-by-default, `ExternalRecipientBlocked`/`RunCapExceeded`, match por domínio (não substring), contador com `Mutex`, `production?` fail-closed, `DEFAULT_TEST_DOMAINS` (test.<domain> + TLDs RFC 2606/6761) e o initializer registrando **sem condição de ambiente**; (3) specs RSpec + variante Minitest esperando o `raise_error` com `ActionMailer::Base.deliveries` vazio, mais spec do cap e do env desconhecido. Tabela regra→código, nota sobre `deliver_later`/Sidekiq e extensão a SMS/push/PSP.
- **references/iam.md**: bullet do Email OTP agora explicita "ligado ≠ entregando pra fora" (sink + interceptor); item novo no checklist de DoD do IAM com a prova exigida.
- **references/testes.md §22**: item obrigatório — teste nunca dispara efeito externo real (`:test`, endereço `@test.<domain>` inclusive na `sequence` do FactoryBot e no `db/seeds.rb`, spec vermelho do guard e do cap, `WebMock.disable_net_connect!`, grep de CI contra `@gmail.com`/terceiros).
- **assets/CLAUDE.md**: piso 17 com o recorte Rails (delivery_method por ambiente, interceptor, cap, fail-closed, rota nula) e o motivo (queima de IP e domínio derruba o OTP de produção).

## [0.1.2] — 2026-08-18
Correção da contradição do muro pré-login de IAM (alinha ao `iam.md` da schematize-engineering).
### Mudado
- **/ruby-iam**: removido o "2º fator forte obrigatório antes do acesso pleno" e o "força 2º fator no 1º login" — o muro pré-login / deadlock de bootstrap VETADO pela norma. Agora senha+Email OTP = 2FA baseline; fator forte é nudge + step-up just-in-time.

## [0.1.1] — 2026-08-18
Q.A. repointado para a skill dedicada **schematize-qa**.
### Mudado
- **`/ruby-qa` virou wrapper fino** da **schematize-qa** (`/qa-plan` → `/qa-run`) no recorte Ruby (`rspec`/RSpec). Referências ao antigo **§22.9** removidas de `SKILL.md`, `references/testes-execucao.md`, `assets/CLAUDE.md` e `/ruby-help`.

## [0.1.0] — 2026-08-15
Primeira release da skill **schematize-ruby** — recorte Ruby dos Padrões de Engenharia
da Casa (linguagem **sancionada** pelo rol: prototipagem rápida, scripts/automação, DX de
produto com Rails e manutenção de legado Ruby).

### Adicionado
- **Corpo normativo fatiado em `references/`**, especializado para Ruby: `arquitetura.md`
  (rol sancionado + fit de Ruby, camadas/DDD, packwerk, estrutura de pastas Rails/Sinatra),
  `padroes-codigo.md` (arquivo ≤750, uma classe/módulo por arquivo, YARD, `frozen_string_literal`,
  RuboCop, service objects, RBS/Sorbet), `seguranca.md` (ActiveRecord parametrizado, strong
  params, `SecureRandom`, `bcrypt`/`argon2`, CSRF, brakeman, YAML seguro), `iam.md` (auth como
  app Ruby separada; Devise sozinho não é o IAM da casa; ≥2 fatores, ReBAC multi-tenant),
  `cadeia-suprimentos.md` (Gemfile.lock, bundler-audit, SBOM), `dados-eventos.md` (ActiveRecord/
  Sequel, migrations reversíveis, Sidekiq, outbox), `testes.md` + `testes-execucao.md` (RSpec/
  Minitest, FactoryBot, SimpleCov, VCR/WebMock — testes load-bearing pela dinâmica do Ruby),
  `concorrencia.md` (GVL, threads, Puma/forking, Ractor experimental, async/fibers, Sidekiq),
  `observabilidade.md` (OTel Ruby, lograge/semantic_logger), `stack-versoes.md` (Ruby/Rails +
  EOL), `operacao.md`, `ops.md`, `entrega.md`, `anti-padroes.md`, `contexto-claude-code.md`.
- **Comandos** (prefixo `ruby-`): `/ruby-help`, `/ruby-load`, `/ruby-claude`, `/ruby-cc`,
  `/ruby-handoff`, `/ruby-qa`, `/ruby-review`, `/ruby-iam`, `/ruby-index`, `/ruby-ops`.
- **Assets**: `CLAUDE.md` sempre-on Ruby-especializado (com o rol de linguagens sancionado),
  templates (ADR/TASK/CHAT_ARCHIVE/PR/RUNBOOK/INDEX_*/MAPA), `settings.claude.example.json`,
  CI, lint (`.rubocop.yml`), hooks de pre-commit.
- **Scripts**: andaime de testes (`lib.sh`, `test-skeleton.sh`, `smoke-selfcheck.sh`,
  `simulated/run.py`), gate de diff, índice de microfunções e hooks de contexto.
- **Instalador** `install.sh` + manifesto `skill.toml` (`slug="ruby"`, `Backend · Ruby`, beta).

### Normativo coberto
- **Escolha de linguagem pelo rol sancionado + fit** (`arquitetura.md` §3): Ruby é permitido
  para serviço novo por **fit + ADR** (não é legado-only); Node-backend e PHP são legado que
  não recebe serviço novo; nova linguagem fora do rol exige ADR de exceção.
- Pisos agnósticos integrais especializados a Ruby: segredo nunca no cliente, ActiveRecord
  parametrizado, auth server-side, IAM por desenho (app separada), testes de verdade + pentest
  de rejeição, archive obrigatório (§28), DoD (§35), índice/MAPA (§39), fluxo de ambientes/ops,
  observabilidade, tempo do usuário acima de tokens.
- Rigor extra de **testes + tipagem (RBS/Sorbet)** para carregar a segurança que o compilador
  não dá, dada a natureza dinâmica do Ruby.
