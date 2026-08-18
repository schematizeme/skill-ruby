# Changelog — schematize-ruby

Formato: [Keep a Changelog]; versionamento: SemVer.

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
