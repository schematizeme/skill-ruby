# Testes — Execução, Pentest e Q.A.

> Parte da skill **schematize-ruby**. Continuação de `testes.md` (numeração **preservada**, §22.4+): padrão de script de teste, seeds/personas, integração com CI, pentest (§22.7–22.8) e o Rakefile padrão (§23). O fluxo de Q.A. plan-first agora mora na skill dedicada **schematize-qa** (`/qa-plan` → `/qa-run`).

---

### 22.4 Padrão de script (`tests/<mode>/<name>.sh`)

O sistema vivo é orquestrado por Bash + `tests/lib.sh` do andaime (`<project>_ops`); o teste **por código** é RSpec/Minitest dentro de cada serviço. Skeleton obrigatório do script de sistema:

```bash
#!/usr/bin/env bash
# <Categoria> · <Nome curto>
# Descrição clara do que cobre e o esperado.
# Esperado: status X em caso Y. Falha = significado Z.

set -uo pipefail
_DIR="$(cd -P "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
# shellcheck source=../lib.sh
source "$_DIR/lib.sh"

test_section "<Categoria> · <Nome>"

API="$(api_base)"

# Caso 1
assert_http_in "<descrição do caso>" "200|401" GET "$API/v1/<rota>"

# Caso 2 — body shape
http_call GET "$API/v1/<rota>"
if [[ "$HTTP_CODE" == "200" ]]; then
  if echo "$HTTP_BODY" | jq -e '.data | length > 0' >/dev/null 2>&1; then
    test_pass "<rota> retorna dados"
  else
    test_fail "<rota> sem 'data'" "$HTTP_BODY"
  fi
fi

test_summary "<categoria>/<nome>"
exit $TEST_EXIT_CODE
```

**MUST**
- Banner do `test_section` na primeira linha do output.
- Cada caso assertado vira uma linha `✓` ou `✗` ou `○ skip`.
- `test_summary` no fim agrega contadores pra o runner.
- Falhas incluem `HTTP_BODY` truncado nos primeiros 2000 chars (sem PII).
- O teste por código correspondente é um `spec` RSpec (`spec/**/*_spec.rb`) rodado com `bundle exec rspec` — o script de sistema **não substitui** o spec, complementa (fronteira viva × unidade).

---

### 22.5 Seeds e personas de teste

**MUST**
- Personas declaradas em `tests/seeds/test-*.rb` (FactoryBot) ou `tests/<mode>/*.json` — versionadas, reproduzíveis. `db/seeds.rb` semeia o ambiente base; fábricas semeiam os casos.
- Senhas test-only **explicitamente flagadas** (ex.: prefixo `SimTest!`) e **rejeitadas em produção** por validador de senha fraca.
- Setup idempotente: `find_or_create_by`/`upsert` — re-rodar não duplica.
- Cleanup automático no fim de cada run? **Não** — deixa lixo pra inspeção. Limpe via comando explícito (`<project> test clean-seeds`).

**Personas mínimas pra cobrir RBAC + multi-tenancy**:

1. `superadmin` — platform role, vê tudo.
2. `tenant_admin_A` — escopo do tenant A.
3. `tenant_admin_B` — escopo do tenant B (pra testar isolamento).
4. `normal_user` — sem role, só dados próprios.
5. (Opcional) `viewer`, `editor`, `support` — combinações de roles dentro de A.

---

### 22.6 Integração com CI

**MUST**
- PR check: `<project> test smoke + security + authz + hardening` (rápido — ~3min total).
- Pré-deploy: `<project> test all` (full minus chaos+unit — ~6min).
- Nightly: `<project> test full` (inclui chaos + unit).
- Bloqueio de merge: fail no `summary.json` (`.totals.fail > 0`) trava merge.
- **Toolchain de CI Ruby, sempre via `bundle exec`:** `bundle exec rspec` (testes), `bundle exec rubocop` (lint/estilo — não negocia com `--auto-gen-config` pra silenciar), `bundle exec brakeman -q -w2` (SAST Rails), `bundle-audit check --update` (CVE em gems). Qualquer um vermelho trava.
- **Ruby não-EOL.** O CI roda na versão de Ruby travada em `.ruby-version`/`Gemfile` e essa versão **não pode estar EOL** (liga com `references/stack-versoes.md`). Rodar em Ruby fora de suporte é achado de supply-chain, não "detalhe de infra".

**Dashboards**
- `summary.json` é parseado por job de métricas pra empurrar `tests_pass_total`, `tests_fail_total`, `tests_duration_seconds_bucket` pra Prometheus.
- Cobertura do SimpleCov (`coverage/.last_run.json`) vira métrica; queda abaixo do piso (§22) trava.
- Falha em smoke pós-deploy dispara rollback automático.

---

### 22.7 Pentest leve via ferramentas externas (complementar)

**SHOULD** — em CI noturno, não a cada PR:
- **brakeman** (`brakeman -q -w2`) — SAST específico de Rails (SQLi, mass-assign, XSS de `raw`, `send_file`, redirect aberto). Roda a cada PR (é rápido e preciso), não só noturno.
- **bundler-audit** (`bundle-audit check --update`) — CVE conhecido nas gems do `Gemfile.lock`.
- **OWASP ZAP baseline** (`zap-baseline.py -t https://api.example.com`) — passive scan.
- **Nuclei** (`nuclei -u https://api.example.com -t cves,exposures,misconfiguration`).
- **trivy fs** ou **grype** sobre o repo + imagens Docker pra CVE em deps + base images.
- **Semgrep** ruleset `p/owasp-top-ten` + `p/ruby`/`p/rails` pra SAST adicional.

Resultados em PR comment ou dashboard, **não bloqueiam merge** por default (alto ruído) — mas qualquer **Critical** sem ADR de aceite trava. Achado do brakeman/bundler-audit de severidade alta **trava** (baixo ruído, alta precisão).

---

### 22.8 Diretrizes de Pentest (princípios, não só scripts)

Os scripts da §22.3 são a implementação; estes são os princípios que regem **como** se faz pentest interno e **o que conta como passar**. Ligam com **schematize-pentest** (metodologia completa, RoE, arsenal por classe).

**Postura**
- **Assume-breach / hostil por padrão.** Todo input vindo de fora (body, query, header, cookie, path, arquivo, webhook) é tratado como hostil até prova de validação. Em Ruby isso inclui **desserialização**: `YAML.load`, `Marshal.load`, `Oj` em modo inseguro sobre dado externo = RCE presumida até prova de `safe_load`.
- **Cobrir a rota toda, não a amostra.** Pentest roda contra o inventário completo de rotas (mesmo catalog do `simulated`, gerado de `rails routes`). Rota nova sem caso de pentest correspondente bloqueia (liga com §35).
- **Caixa-cinza.** O time tem o OpenAPI, o schema e as roles — testa sabendo o que **deveria** acontecer, não no escuro. Pentest sem mapa vira teatro de "tentei o `' OR 1=1` e deu 403, tá seguro".

**Critério de resultado — o que é PASS / FAIL**
- **Nunca `500` por input malicioso.** Erro do servidor (5xx / página de exception do Rails) diante de payload hostil é FALHA — significa que o input chegou fundo demais sem validação. O esperado é `400`/`422` (input inválido) ou `401`/`403`/`404` (sem acesso).
- **Nunca eco sem escape.** XSS/template injection refletido no body/header sem encode = FALHA, mesmo com status 200 (`html_safe`/`raw`/`<%== %>` indevido é a origem).
- **Nunca coerção silenciosa.** `"123".to_i`, string vazia virando `0`, `"true"` virando bool, `params[:id]` string casando com id numérico sem checagem = FALHA (liga com `type-confusion.sh`). O tipo do schema/validação é contrato.
- **Nunca vazamento entre tenants/usuários.** Qualquer `200` com dado de outro escopo = FALHA crítica, para o deploy na hora (§15). Atenção a `unscoped`, `default_scope` esquecido e `Model.find` sem escopo de tenant.
- **Nunca diferença observável que ajude o atacante.** Mensagem de erro distinta pra "user existe" vs "senha errada", timing > 30% de variância, stack trace no response = FALHA.
- **Idempotência sob ataque.** Replay, duplicação, race no mesmo recurso não corrompe estado.

**Severidade e gate**
- Classificar todo achado: `Critical` / `High` / `Medium` / `Low` / `Info`.
- **`Critical` e `High` bloqueiam merge/deploy.** Sem exceção silenciosa — ou conserta, ou ADR de aceite de risco **com prazo de remediação e dono** (e mesmo assim `Critical` de auth/tenant/injeção/desserialização **não aceita ADR** — é piso da §37).
- `Medium`/`Low` viram issue rastreável com prazo; acumular não é opção.

**Higiene**
- **Pentest roda só contra `dev`/`staging`** (ou alvo autorizado). Scripts destrutivos (`service-kill`, `db-disconnect`) gated por env var explícita (§22.3 chaos). Nunca contra produção sem autorização formal e janela.
- **Evidência sempre.** Todo achado tem request/response reproduzível no `raw.jsonl` — "achei uma falha" sem repro não conta.
- **Falso-positivo é bug do teste.** Pentest que grita sem motivo treina o time a ignorar — calibra ou remove. Mas **na dúvida, FAIL** (vai pra `REVIEW`, olho humano decide), nunca silencia por padrão.

**Escopo mínimo coberto** (mapeia OWASP Top 10 + API Top 10): injeção (SQL/NoSQL/command/template/ERB), XSS, validação de tipo e sanitização (§22.3), authz/IDOR/BOLA, broken auth (JWT, sessão, refresh), SSRF, mass assignment (Strong Params), **desserialização insegura (YAML/Marshal)**, exposição excessiva de dados, misconfig (headers, CORS, paths expostos, `config/master.key`), rate limiting, e abuso de recurso (payload gigante, ReDoS `^`/`$`, billion-laughs). Detalhe por classe: **schematize-pentest**.

> Pentest não "tenta hackear pra ver se acha algo". Ele **prova, rota por rota, campo por campo, que o sistema rejeita o que deve rejeitar.** Achado é evidência; ausência de achado só vale se a cobertura for total.

---

### Q.A. — disciplina na skill schematize-qa

O fluxo de execução de Q.A. plan-first (planeja tudo → gera o MD de passo a passo → **pede aprovação antes de rodar** → executa faseado/assistido ou de uma vez com watchdog → coleta `summary.json` e trava nos gates) agora mora na skill dedicada **schematize-qa** (`/qa-plan` → `/qa-run`).

No recorte Ruby, o Q.A. usa as ferramentas da linguagem — `rspec` (RSpec)/`bundle exec rspec`, `FactoryBot`, `Capybara`, `WebMock`/`VCR` — e o wrapper fino `/ruby-qa`. Nada de Q.A. roda sem plano aprovado registrado em `<projeto>_archive/qa/`.

---

---

## 23. Rakefile Padrão

O `Rakefile` é a interface de tarefas do serviço (equivalente ao Makefile da casa). Toda tarefa roda o binário travado no `Gemfile.lock` (`bundle exec`), nunca o global.

```bash
rake dev              # sobe ambiente local (bin/dev / foreman)
rake spec             # bundle exec rspec (unit + integration)
rake spec:unit
rake spec:integration
rake lint             # bundle exec rubocop
rake fmt              # rubocop -A (autocorrect seguro)
rake typecheck        # srb tc / steep check (Sorbet/RBS)
rake build            # gem build / assets:precompile
rake run              # bundle exec puma / rails s
rake docker
rake db:migrate       # ActiveRecord migrations
rake db:seed          # db/seeds.rb + FactoryBot
rake docs             # gera/valida openapi
rake security-scan    # brakeman + bundler-audit (sast + sca local)
rake smoketest
rake pentest-light
rake ci               # tudo que o CI roda: spec + rubocop + brakeman + bundle-audit
rake clean
```

> `bin/` guarda os scripts de entrada versionados (`bin/dev`, `bin/setup`, `bin/rails`) — binstubs do bundler, pra que o clone rode sem instalar nada global. `rake ci` é o mesmo conjunto que o pipeline roda: verde local = verde no CI.

---

---
