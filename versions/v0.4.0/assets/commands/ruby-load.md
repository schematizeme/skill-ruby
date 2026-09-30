---
description: schematize-ruby — carrega à força TODO o corpo normativo (DDD/arquitetura, clean code, segurança, dados, concorrência/GVL, testes, operação) no contexto e passa a aplicá-lo no projeto atual como regra inegociável
---

Carregue **à força** e passe a aplicar **integralmente** os Padrões de Engenharia da Casa (recorte Ruby, skill `schematize-ruby`) neste projeto. A partir de agora, nesta sessão, isto **não é opcional**.

1. **Leia agora, na íntegra, TODOS os arquivos** de references da skill — não trabalhe de memória, abra cada arquivo. O caminho é `.claude/skills/schematize-ruby/references/*.md` (instalação no projeto) ou `~/.claude/skills/schematize-ruby/references/*.md` (instalação global). Com destaque para:
   - `padroes-codigo.md` — **clean code**: arquivo ≤750 linhas (~500 úteis + ~250 comentário; flag >300 úteis), **uma classe/módulo por arquivo**, métodos pequenos (RuboCop `Metrics/*`), doc-comment YARD obrigatório (motivo/comportamento/I-O), `# frozen_string_literal: true`, RBS/Sorbet no não-trivial, `MAPA.md`.
   - `arquitetura.md` — **rol de linguagens sancionado + fit de Ruby** (§3), DDD/camadas, repository pattern isolando ActiveRecord, packwerk, bounded contexts, anti-monólito, estrutura de pastas Rails/Sinatra.
   - `seguranca.md` — ActiveRecord parametrizado (VETADO SQL interpolado), strong params, JWT, multi-tenancy, LGPD, segredo nunca no cliente/bundle, `SecureRandom`, `bcrypt`/`argon2`, CSRF, brakeman, YAML seguro.
   - `iam.md` — **IAM da casa (recorte Ruby)**: auth como **app Ruby separada** (`auth.<domain>`), Devise sozinho não é o IAM, ID≠email, ≥2 fatores (`webauthn`/`rotp`/OTP), ReBAC multi-tenant deny-default, sessão 7d/90d, logout irreversível, migração de legado prioridade 0.
   - `dados-eventos.md` — ActiveRecord/Sequel, migrations reversíveis, Sidekiq/ActiveJob, outbox, cache Redis, APIs, resiliência.
   - `concorrencia.md` — **a GVL**, threads (I/O-bound), Puma/forking e Ractor (CPU-bound), async/fibers, Sidekiq; CPU-bound exige processos/Ractors, não threads.
   - `cadeia-suprimentos.md` — Gemfile.lock, bundler-audit, SBOM, imagem mínima/pinada, `.ruby-version` não-EOL.
   - `testes.md` — RSpec/Minitest, FactoryBot, SimpleCov, VCR/WebMock, "verde de verdade", pentest, Q.A. plan-first. **Testes são load-bearing pela dinâmica do Ruby.**
   - `observabilidade.md` — OTel Ruby, lograge/semantic_logger, healthchecks, performance, FinOps.
   - `operacao.md` + `entrega.md` — config, deploy, git/PR, runbooks, ADR, **archive**, DoD, índice.
   - `ops.md` — **control plane `<projeto>_ops`**: fluxo dev→local→github→hml→prd, ops como interface única, instalação paralela=`nproc`, independência=invariante.
   - `stack-versoes.md` — versões e EOL de Ruby/Rails + limiares de gems (fonte volátil).
   - `anti-padroes.md` — a lista completa de anti-padrões vetados (§37).
   - `contexto-claude-code.md` — gestão de contexto/handoff em sessões longas.

2. **Confirme ao usuário** que leu, com **1 linha por arquivo** resumindo o piso central de cada um.

3. Deste ponto em diante, **aplique estes padrões como regra inegociável** em toda decisão, geração e revisão de código deste projeto. Em conflito entre "fazer rápido" e o padrão, **o padrão vence**.

4. **Atualize o `CLAUDE.md` da raiz** do repositório com a versão atual de `assets/CLAUDE.md` da skill — **sobrescreva mesmo se já existir** (rodar não pode deixar a versão antiga; se houver customização local, backup `./CLAUDE.md.bak` e reaplique por cima; se houver bloco de outra skill, mescle). É o mesmo que o comando `/ruby-claude`. Confirme a versão aplicada.
