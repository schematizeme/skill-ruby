# Testes, Pentest e Rakefile

> **Dividida:** §22.4–§23 (padrão de script, seeds, CI, pentest §22.7–22.8, Q.A. §22.9, Rakefile) estão em `references/testes-execucao.md`. Numeração contínua entre os dois.

> Parte da skill **schematize-ruby**. As referências cruzadas (§N) apontam para seções do corpo completo — todas presentes no conjunto de references desta skill.

## Índice
- 22. Testes
- 23. Rakefile Padrão

---

## 22. Testes

**Obrigatório**
- Testes em **dois eixos**: por código (unit/integration/request/system, dentro de cada serviço) e por sistema vivo (smoke/security/pentest/authz/hardening/chaos/simulated, no repo `<project>_ops`).
- Caminhos críticos (auth, pagamento, autorização, billing, eventos de domínio, multi-tenancy) têm testes explícitos cobrindo sucesso, falha e edge cases — independentemente da cobertura agregada.
- **Teste NUNCA dispara efeito externo de verdade** (e-mail, SMS, push, webhook, cobrança): `delivery_method = :test` no ambiente de teste, `ExternalRecipientGuard` registrado como **interceptor**, destinatário sintético `<papel>+<run-id>-<n>@test.<domain>` (inclusive na `sequence` do FactoryBot e no `db/seeds.rb`), e o **spec vermelho do guard** — `expect { UserMailer.otp(to: "alguem@gmail.com", code: c).deliver_now }.to raise_error(ExternalRecipientGuard::ExternalRecipientBlocked)` com `expect(ActionMailer::Base.deliveries).to be_empty` — mais o spec do **cap por execução**. Código pronto em `references/iam.md` §3.1; normativa em `schematize-engineering` → `references/efeitos-externos.md`. **VETADO** `@gmail.com`/domínio de terceiro/e-mail de pessoa real em `spec/`, `test/`, factories ou seeds (grep no CI trava). `WebMock.disable_net_connect!` ligado fecha a última porta.
- **Frameworks: RSpec (principal) / Minitest.** Rodar sempre via `bundle exec rspec` (nunca `rspec` global — versão presa no `Gemfile.lock`). **FactoryBot** para fábricas de dados (não fixtures gigantes e frágeis). **SimpleCov** para cobertura. **VCR/WebMock** para gravar e **bloquear** I/O externo nas fronteiras. Property-based com **rantly**/**propcheck**; mutation com **mutant** no domínio crítico. Request specs para API, system specs (Capybara) para fluxo de UI.

**A dinâmica do Ruby torna o teste LOAD-BEARING**

Ruby não tem compilador que pegue `NoMethodError`, typo de método, aridade errada ou `nil` inesperado antes de rodar. O que em Go/Rust o compilador barra, em Ruby **só o teste (mais a tipagem gradual RBS/Sorbet) barra**. Por isso teste aqui não é rede de segurança opcional — é a **camada de verificação que substitui o compilador**:
- **Cobertura de caminho de erro é piso, não meta.** `NoMethodError`/`nil`/`ArgumentError` estourando em produção = **teste faltando**, sempre — não é "erro raro", é caminho não exercido.
- Cada método público do domínio tem teste que o **executa de fato** (não só o carrega): sucesso, entrada inválida, `nil`, tipo errado, coleção vazia.
- Tipagem gradual (**RBS**/**Sorbet**) cobre o que o teste não alcança em estático; teste cobre o que a tipagem gradual não garante em runtime. Os dois juntos são o piso — nenhum sozinho.

**Cobertura mínima (por código)**

| Camada | Mínimo |
|---|---|
| `domain` / `models` (regra de negócio) | 80% |
| `application` / `services`, `interactors` | 70% |
| `infrastructure` / `adapters`, gateways | 40% |
| Global | 60% |

> Line coverage no Ruby é ainda **mais enganoso** que em linguagem compilada: uma linha "coberta" pode ter passado por um branch que nunca exercitou o `nil`. Editar o threshold do SimpleCov ou pular exemplo pra "passar o CI" é VETADO (§37). O número é contrato.

**Mutation testing (SHOULD → MUST no domínio crítico)** com **mutant**: se o teste não mata a mutação, o teste é decorativo. Line coverage prova que a linha rodou; mutant prova que o teste **verifica** o que a linha faz. Em Ruby, onde não há compilador, mutant é o gate mais próximo de garantia.

---

### 22.1 Test Kit do `<project>_ops`

Toda a malha de testes "do sistema vivo" mora num repo dedicado (`<project>_ops`), invocada por um CLI único.

**Estrutura padrão**

```
<project>_ops/
├── bin/
│   └── <project>-test          # CLI: run modes, agrega saída
├── tests/
│   ├── lib.sh                  # helpers compartilhados (cores, assertions, http_call)
│   ├── README.md               # tabela de modos × scripts × duração
│   ├── smoke/                  # health, rotas-chave, shape de respostas
│   ├── integration/            # login real + CRUD ponta-a-ponta
│   ├── security/               # auth bypass, headers, rate-limit
│   ├── pentest/                # OWASP + extensão
│   ├── authz/                  # RBAC + multi-tenancy isolation
│   ├── hardening/              # TLS, cookies, CORS, exposed paths
│   ├── chaos/                  # fuzz, property-based, kill, db-disconnect
│   ├── simulated/              # matriz exaustiva: rotas × personas × injections
│   ├── seeds/                  # SQL/FactoryBot de setup (personas de teste, superadmin)
│   ├── unit/                   # delega `bundle exec rspec` por serviço
│   └── workspace-code.sh       # smell-scan no monorepo (arquivos > N linhas, etc.)
├── Rakefile
└── README.md
```

**CLI padrão**

```bash
<project> test                  # default: smoke
<project> test smoke
<project> test integration
<project> test security
<project> test pentest
<project> test authz
<project> test hardening
<project> test chaos
<project> test simulated
<project> test unit
<project> test all              # tudo exceto chaos+unit
<project> test full             # all + chaos + unit
<project> test seed-superadmin  # setup inicial (uma vez)
```

**MUST**
- Cada script é executável standalone: `bash tests/smoke/auth-endpoints.sh` deve rodar e sair `0/1`.
- Cada script declara `TEST_NAME` e usa helpers de `lib.sh` (`test_pass`, `test_fail`, `test_skip`, `test_section`, `test_summary`, `http_call`, `assert_http_in`).
- Banner ENORME em vermelho quando há falha — feedback visual impossível de ignorar em CI.
- Skip sem erro quando dependência opcional falta (ex.: openssl ausente em hardening/tls).

---

### 22.2 Saída estruturada (machine-readable)

Toda execução escreve em `/<project>/logs/test-<YYYY-MM-DD>-<HHMMSS-pid>/`:

| Arquivo | Conteúdo |
|---|---|
| `summary.txt` | PASS/FAIL por script, legível |
| `summary.json` | **fonte única pra dashboards/CI** — schema fixo abaixo |
| `run-totals.txt` | 7 linhas: mode, started, finished, duration, scripts, pass, fail, exit |
| `<mode>-<name>.log` | output completo de cada script |
| `cookies.txt` | sessão dos personas (só em `integration`) |
| `coverage-summary.{txt,json}` | cobertura por serviço via SimpleCov (só em `unit`) |

**Schema `summary.json`**

```json
{
  "started_at": "2026-05-11T11:35:50Z",
  "finished_at": "2026-05-11T11:35:57Z",
  "duration_seconds": 7,
  "mode": "smoke",
  "log_dir": "/<project>/logs/test-2026-05-11-083550-448694",
  "scripts": [
    {"category": "smoke", "name": "auth-endpoints", "status": "pass"},
    {"category": "smoke", "name": "services-health", "status": "fail"}
  ],
  "totals": {"pass": 23, "fail": 1, "scripts": 24, "exit_code": 1}
}
```

**MUST**
- Exit code 0 se tudo passou; 1 se qualquer caso falhou.
- `summary.json` é contrato — não quebre os campos `mode`, `totals`, `scripts[]`.
- RSpec emite JSON parseável no unit: `rspec --format json --out rspec.json` (mais o `progress` humano). Reconcilie contra o `summary.json`.
- Logs zipados e enviados pra storage de longo prazo após cada CI run (90 dias mínimo).

---

### 22.3 Categorias de teste (por modo)

#### `smoke` — saúde do sistema, < 2min

Cobre: health/metrics de cada serviço, rotas-chave do dispatcher, frontends, observability stack, endpoints por domínio (auth, catalog, billing, etc.), DB connectivity, microservices-status, shape de health/metrics, response-time p95, CORS preflight, OpenAPI availability, log scan pra PII vazada.

Falha = bloqueio de deploy. Roda **antes e depois** de cada `update` em todo ambiente.

**MUST — anti "verde mentiroso" (smoke que passa com bug dentro)**

Um smoke que só confere status `200` é teatro: a rota responde, o conteúdo está quebrado, e o deploy passa. Em Ruby o risco é maior — um `render json:` de um objeto meio-construído devolve `200` com `null` no campo. Para impedir isso:

- **Assertar conteúdo, não só status.** Toda rota-chave valida o **shape do body** (campos esperados via `jq -e`), não apenas o código HTTP. `200` com body vazio, `{}`, `null`, `[]` onde deveria haver dado, ou HTML de erro (página de exception do Rails) com status 200 = **FALHA**.
- **Assertion negativa obrigatória.** Cada rota crítica também testa que o que **não** deveria estar lá não está: sem stack trace / `ActionView::Template::Error`, sem `error`/`exception` no body de sucesso, sem `"nil"`/`"null"`/`"#<...>"` (inspect de objeto vazado) serializado, sem placeholder de template não renderizado (`<%=`, `{{`, `#{`).
- **Self-test do próprio smoke (meta-teste).** O suite tem um caso que **força uma falha conhecida** (ex.: bater numa rota fake `/_smoke_canary_should_404` esperando 404, e uma asserção que deve falhar de propósito num modo `--self-check`) pra provar que o runner **consegue reportar FAIL**. Se o "self-check" passa quando deveria falhar, o smoke está cego → CI quebra. Nenhum teste pode ser estruturalmente incapaz de falhar.
- **Cobertura de rota verificada.** O smoke compara as rotas que testou contra o inventário do OpenAPI/`rails routes`. **Rota em produção sem caso de smoke = FALHA**, não silêncio. (Liga com §35 e com `simulated`.)
- **Sem `|| true`, sem swallow.** Proibido `curl ... || true`, `set +e` sem `set -e` de volta, ou condição que transforma erro em pass. Falha de rede/timeout numa dependência obrigatória é FAIL, não skip (skip só pra dependência **opcional** declarada).
- **Latência e dado fresco.** Healthcheck que devolve `200` cacheado/estático não conta — `/ready` valida dependência **de verdade** (ping no DB via `ActiveRecord::Base.connection.active?`, no Redis, no broker), e o smoke afere `response-time p95` contra o SLO (§30). Resposta lenta demais = FALHA.
- **Fail loud.** Banner vermelho ENORME (§22.1) e o `summary.json` com `totals.fail > 0` travando o deploy. Verde só quando **todas** as asserções de conteúdo passaram.

> Smoke que nunca falha não é smoke saudável — é smoke quebrado. Se você não viu o teste falhar de propósito, você não sabe se ele funciona.

#### `integration` — fluxos end-to-end com credenciais reais, < 3min

Cobre: login do superadmin com cookie/sessão real, CRUD course/user/subscription via API admin, upload de imagem, reset de senha completo. Usa seed `tests/seeds/test-superadmin.rb` (FactoryBot) pra garantir user existente. Request specs do RSpec batem na app real; VCR grava as chamadas a terceiros pra o teste **não** depender de rede viva.

**Requer pré-seed**: rodar `<project> test seed-superadmin` na primeira vez no ambiente.

#### `security` — controles de segurança "óbvios", < 1min

Cobre: auth bypass tentando rotas admin sem cookie, headers de segurança (CSP, X-Content-Type, HSTS), rate limit no login (50 paralelas → expect 429; `rack-attack` configurado), formato de password hash (bcrypt cost ≥ 12 / argon2id), `bundle-audit check --update` (CVE em gems), `brakeman -q` (SAST Rails), JWT algorithm validation (RS256 only).

#### `pentest` — OWASP Top 10 + extensão, < 3min

Scripts cobrem (cada um isolado):
- `sql-injection.sh` — payloads SQLi clássicos em query/body. Esperado: 400/422 ou 401/403/404. **NUNCA 500** e nunca 200 revelando "OR 1=1". (Atenção: `where("name = '#{x}'")` e `find_by_sql` com interpolação são a porta — o teste tem que provar que morreram.)
- `xss.sh` — `<script>`, `<img onerror>`, `javascript:` em campos refletidos. Resposta não pode incluir payload sem escape (`html_safe`/`raw` indevido no ERB = achado).
- `idor.sh` / `user-bola.sh` — IDs de outro tenant/user nos paths (o clássico `Model.find(params[:id])` sem escopo de tenant).
- `ssrf.sh` — URLs apontando pra `127.0.0.1`, `169.254.169.254`, `file://`, redirect chains (`open-uri`/`Net::HTTP` sem allowlist).
- `jwt-tampering.sh` / `jwt-claim-tampering.sh` / `jwt-algorithm.sh` — alg=none, alg=HS256-com-RS256-pubkey, exp futuro, sub trocado.
- `path-traversal.sh` — `../`, `..%2f`, null bytes (`send_file`/`File.read` com path do usuário).
- `mass-assignment.sh` — POST com campos extras (`is_admin`, `tenant_id`, `created_at`) — prova que Strong Parameters (`params.permit`) barram; `permit!` ou `params.to_unsafe_h` = achado.
- `open-redirect.sh` — `?next=https://evil.com` (`redirect_to params[:next]`).
- `host-header.sh` — Host header arbitrário pra envenenar `url_for`/links em emails.
- `csrf.sh` — POST sem token CSRF válido (`protect_from_forgery` ativo?).
- `http-method-tampering.sh` — TRACE, OPTIONS, métodos não suportados.
- `request-smuggling.sh` — `Transfer-Encoding: chunked` + `Content-Length` conflitantes.
- `cache-poisoning.sh` — headers exóticos que envenenam CDN.
- `cookie-bomb.sh` — flood de cookies grandes → expect 4xx limpo, não 500.
- `session-fixation.sh` — session ID atribuído pré-login persiste pós-login (`reset_session` no login?).
- `timing-attack.sh` — diff de latência entre user existente vs inexistente no login (alvo: < 30% variance; usar `secure_compare`).
- `mass-assign-deserialize.sh` — YAML/Marshal em campo de entrada (`YAML.load`/`Marshal.load` de dado externo = RCE; tem que ser `YAML.safe_load`).
- `redos.sh` — strings catastróficas pra regex (`aaaaaa...aaa!`); atenção a `^`/`$` no Ruby (usar `\A`/`\z`).
- `billion-laughs.sh` — JSON/XML bomb (nested arrays profundos; Nokogiri sem `NONET`/entidades).
- `clickjacking.sh` — `X-Frame-Options` / CSP `frame-ancestors`.
- `refresh-token-reuse.sh` — usar mesmo refresh duas vezes → 2ª deve revogar família.
- `parameter-pollution.sh` — `?id=1&id=2` (Rack agrupa em array — o controller espera escalar?).
- `excessive-data-exposure.sh` — endpoint público vaza email/cpf/telefone (`to_json` sem `only:`/serializer).
- `header-spoof.sh` — `X-Forwarded-For`, `X-Real-IP`, `X-Original-User` injetados.
- `oversized-payload.sh` — body de 10MB+ → 413, não 500.

**Validação de tipo e sanitização de entrada** — campo só aceita o que deveria, e o que não deveria vira **422/400 limpo, nunca 500 e nunca persistido cru**. Em Ruby (tudo é objeto, `params` são strings) a coerção silenciosa é o bug número um:
- `type-confusion.sh` — manda o tipo errado em cada campo: string onde o schema espera `integer`/`uuid`/`bool`/`enum`/`date`; número onde espera string; array onde espera hash; hash aninhado onde espera escalar. Esperado: `422` com erro de validação. **Aceitar `"123"` como int por `.to_i` silencioso (que vira `0` em lixo!), ou estourar 500, é FALHA.**
- `boundary-values.sh` — limites numéricos: negativo onde só positivo, `0`, `MAX_INT+1` (Ruby tem Bignum — não estoura, mas o banco `integer` estoura), `-1`, float onde espera int, `NaN`, `Infinity`, notação científica (`1e999`).
- `charset-fuzz.sh` — caracteres estrangeiros e estranhos em **todo** campo de texto: unicode astral (emoji 𝕏, `𝓪`), CJK (中文), árabe/hebraico (RTL), combinação de diacríticos, zero-width (`​`), homoglyphs, ` ` null byte, control chars (`\x01`-`\x1f`), BOM, encoding inválido (`ASCII-8BIT` cru → `Encoding::CompatibilityError`). Esperado: aceitar normalizado (NFC via `unicode_normalize`) **ou** rejeitar com 422 — **nunca** quebrar encoding, corromper o dado, ou refletir sem escape.
- `format-validation.sh` — campos com formato declarado (email, cpf, telefone, url, uuid, cep) recebem lixo que casa o "shape" mas é inválido (`a@b`, cpf com dígito verificador errado, `uuid` de 35 chars). Validação semântica no model (`validates ..., format:` + validação de negócio), não só regex frouxa.
- `length-overflow.sh` — string acima do limite, campo obrigatório vazio/ausente/`nil`, whitespace-only. Esperado: 422, e o limite **vem da validação declarada** (`validates :x, length: { maximum: n }`), não de um `varchar(255)` implícito que estoura no banco com `ActiveRecord::ValueTooLong`.
- `injection-in-every-field.sh` — roda o conjunto SQLi + XSS + path-traversal + command-injection (`system`/`\``/`Open3` com input) + template-injection (`ERB.new(user_input)`, `#{}`, `{{7*7}}`) contra **cada** parâmetro de **cada** rota mutável, não só os "óbvios". Tudo que entra é tratado como hostil até prova de sanitização.

> Regra do pentest de entrada: **todo campo é um campo de ataque.** Se o model diz `integer`, prove que `integer` é tudo que entra — e que `.to_i` não engoliu lixo virando `0`. Se diz texto, prove que sai escapado e normalizado. Coerção silenciosa e `500` são as duas faces do mesmo bug.

#### `authz` — autorização e isolamento, < 1min

- `cross-tenant-idor.sh` — tenant A não vê dados de tenant B (cobre §15: `default_scope`/escopo explícito com `tenant_id` sempre no WHERE — cuidado com `unscoped`).
- `rbac-negative.sh` — viewer não consegue write, editor não consegue admin (Pundit/CanCanCan negando).
- `privilege-escalation.sh` — tenant_admin não consegue virar platform superadmin.
- `permission-boundary.sh` — combinações de roles + recursos → matriz de allow/deny.
- `tenant-isolation.sh` — cookie/JWT de tenant A injetado em rota de tenant B → 403.

#### `hardening` — endurecimento da superfície de ataque, < 1min

- `tls-config.sh` — TLS 1.2/1.3 ok, 1.0/SSLv3 rejeitados, cert válido, HSTS no header, nome bate.
- `headers-full.sh` — CSP, COOP, CORP, Referrer-Policy, Permissions-Policy, X-Content-Type-Options (`secure_headers`/config do Rails).
- `cookies.sh` / `cookie-attributes.sh` — HttpOnly + Secure + SameSite=Lax|Strict em todos os cookies de sessão.
- `cors.sh` — origens permitidas explícitas (`rack-cors`), sem `*` em rotas autenticadas.
- `exposed-paths.sh` — `.git/HEAD`, `.env`, `config/master.key`, `/rails/info`, `phpinfo.php` retornam 404 (não 200).
- `default-creds.sh` — login com `admin/admin`, `root/root`, `test/test` falha sempre.

#### `chaos` — comportamento sob stress / falha, < 5min

- `input-fuzz.sh` — strings longas (10MB), null bytes, unicode astral, JSON malformado.
- `property-based.sh` — idempotência (`POST` com `Idempotency-Key` igual 2x → mesmo resultado), p95 < SLO, JWKS sempre formado corretamente, enums com valores fora do range (rantly/propcheck).
- `service-kill.sh` — `systemctl stop <service>`, mede recuperação (gated por `EDUCE_CHAOS_ALLOW=1` — perigoso).
- `db-disconnect.sh` — derruba conexão DB momentaneamente, valida que o pool do ActiveRecord recupera (`reconnect`/checkout).
- `concurrent-load.sh` — 50 GETs concorrentes em rotas públicas, mede taxa de sucesso e p50/p95/p99, hang > 10s = falha (atenção ao pool de threads do Puma vs pool de DB — ver `references/concorrencia.md`).

#### `simulated` — matriz exaustiva, ~5min

Engine Python (`tests/simulated/run.py`) que cruza **rotas × personas × injections**:

- **Personas (mínimo 3)** declaradas em `personas.json`:
  - `superadmin` (platform role)
  - `tenant_admin` (escopo de 1 tenant de teste)
  - `normal_user` (sem roles)
  Cada persona declara `expected_access` por categoria de rota (`public`, `auth`, `authenticated`, `admin`, `internal`, `import`).

- **Injections (mínimo 10)** em `injections.json`:
  - SQLi (`' OR 1=1 --`, `'; DROP TABLE users CASCADE; --`)
  - XSS (`<script>alert(1)</script>`)
  - Path traversal (`../../etc/passwd`, `..%2f..%2fetc%2fpasswd`)
  - Null byte, unicode RTL override, long string
  - Mass-assignment keys (`is_admin`, `tenant_id`, `created_at`, `password_digest`)
  - Deserialization (`--- !ruby/object`, YAML/Marshal payload)

- **Rota catalog** — JSON gerado a partir do OpenAPI ou de `rails routes --expanded` (rotas + controller#action + verbos).

**MUST — cobertura total de rotas (garantia de acessibilidade)**
- O engine **enumera 100% das rotas** do catalog e prova, por persona, que cada uma responde como esperado (acessível pra quem deve, `403`/`401` pra quem não deve). **Rota no catalog sem resultado no `raw.jsonl` = FALHA** — não existe rota "não testada".
- **Reconciliação obrigatória:** rota servida em runtime mas ausente do catalog (rota fantasma) **e** rota no catalog que não responde (rota morta / `404` inesperado) **ambas** quebram o run. O número de rotas testadas tem que bater com o `rails routes`.
- Toda rota é exercida com persona autorizada **e** não autorizada — acessibilidade e isolamento no mesmo passe.
- Saída lista explicitamente, no `report.md`, a **matriz rota × persona × esperado × obtido**, com as linhas `REVIEW` destacadas pra olho humano.

**Outputs**:
- `raw.jsonl` — uma linha por request, toda evidência.
- `report.md` — relatório humano com seções **AUTO** (passou claro) e **REVIEW** (status inesperado, precisa olho humano).
- `summary.json` — totais por categoria.

**Setup**: `bash tests/simulated/setup.sh` cria as 3 personas no DB via FactoryBot com hash bcrypt.

Vars de ambiente úteis (`EDUCE_TEST_*` no Educe — generalizar com `<PROJECT>_TEST_*`):
- `<P>_TEST_API_BASE` (default `http://127.0.0.1:13000`)
- `<P>_TEST_LOG_DIR` (default `/<project>/logs`)
- `<P>_SIM_TENANT_ID`
- `<P>_SIM_MAX_ROUTES` (debug: limita)
- `<P>_SIM_SKIP_MUTATIONS=1` (só GET)

#### `unit` — testes por código, 5-15min — **agressivos, não decorativos**

CLI delega para o runner de cada serviço:
- Ruby/RSpec: `bundle exec rspec` (com SimpleCov ligado)
- Ruby/Minitest: `bundle exec rake test`
- Front (se houver): `vitest run` / `jest`

Gera `coverage-summary.json` agregando cobertura por serviço via SimpleCov (`--merge`).

**MUST — teste unitário que realmente caça bug (não só cobre linha)**
- **Caminho de erro é obrigatório, não opcional.** Para cada método, testar o sucesso **e** as falhas: input inválido, dependência que levanta exceção, timeout, `nil`, array/hash vazio, divisão por zero. Em Ruby, o `nil` que vira `NoMethodError` na linha seguinte é *a* falha clássica — cobrir só o happy-path é cobertura mentirosa.
- **`NoMethodError`/aridade/typo pego pelo teste, não pela produção.** Sem compilador, o exemplo que exercita o método é a única prova de que ele existe e recebe o que espera. Método público sem exemplo que o chame de verdade = buraco.
- **Tabela de casos hostis por validador/parser:** tipo errado, fora do range, string gigante, vazia, unicode/RTL/null byte, número como string e vice-versa (`"1abc".to_i == 1` é armadilha) — espelhando o `type-confusion`/`charset-fuzz` do pentest, só que na fronteira do método. Bug de sanitização morre no unit, antes do pentest achar.
- **Property-based testing (SHOULD → MUST em domínio crítico):** rantly/propcheck. Idempotência, round-trip (`serialize`→`deserialize`), invariantes de agregado. Encontra o edge case que você não imaginou.
- **Mutation testing no domínio crítico** (§22, **mutant**): se o teste não pega a mutação, o teste é decorativo. **Mutation score mínimo definido por serviço crítico**, não só line coverage.
- **VCR/WebMock obrigatório na fronteira externa:** teste que bate rede real é flaky e mente. Grave a interação uma vez (VCR cassette), bloqueie qualquer HTTP não registrado (`WebMock.disable_net_connect!`). Cassette com segredo é limpo antes de commitar.
- **Boundary obrigatório:** `0`, `-1`, `1`, `MAX`, vazio, `nil`, um, muitos. O bug mora na borda.
- **Proibido teste que não pode falhar:** exemplo sem `expect`, `expect(true).to be_truthy`, mock (`allow`/`double`) que devolve exatamente o esperado sem exercer nada. Revisão de PR rejeita teste tautológico. Cuidado com o mock que esconde o `NoMethodError` real — verificar contra o objeto verdadeiro (`instance_double`, não `double` solto).

> Cobertura mede o que o teste **executa**, não o que ele **verifica**. Mutation testing (mutant) mede o que ele verifica. Em Ruby, onde o compilador não existe pra pegar o typo, isso não é refinamento — é o piso. Por isso line coverage é piso, não meta.

---
