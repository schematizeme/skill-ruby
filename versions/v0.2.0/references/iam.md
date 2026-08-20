# IAM — Identidade e Autorização da casa (piso inegociável) — recorte backend/Ruby

Piso normativo de **identidade, autenticação e autorização**, válido para toda skill
da casa. **Todo projeto começa com um IAM robusto por desenho** — segurança é
inegociável. Este arquivo é o recorte **backend/Ruby** do modelo agnóstico
(`schematize-engineering/references/iam.md`): mantém **todas as regras normativas** e
as concretiza no serviço de auth como **aplicação Ruby separada** (`<projeto>_auth_rb`,
Rails ou Sinatra).

Princípios-âncora: **separar identidade de autorização**; **nunca menos de 2 fatores**
(senha + Email OTP **já conta** como 2FA baseline — fator forte é incentivado e just-in-time,
**nunca muro pré-login**); **força adaptável ao risco** (2FA→3FA + negação deceptiva sob
suspeita); **recuperação tão forte quanto o login**; **deny-by-default**; **enforcement sempre
no servidor**. O buraco clássico é ter 2FA no login e um reset por 1 email que passa por
cima — aqui isso é vetado.

## 1. Topologia — auth é uma APLICAÇÃO SEPARADA (serviço Ruby próprio)

- **A autenticação é um serviço próprio, com link próprio e front próprio**, servido em
  **`auth.<domain>`**. **VETADO** apensar o auth ao escopo principal como monolith (nem
  como um `app/controllers/auth` do serviço principal, nem como engine Rails montada no
  host).
- **Serviço de auth em Ruby** (`<projeto>_auth_rb`, Rails ou Sinatra) + **front de auth**
  próprio (`<projeto>_authfront`), com **repo, deploy, user Linux e systemd/container
  isolados** por conta própria (casa com o isolamento por app do `ops.md` §3). Comprometer
  o app principal **não** compromete o IdP. O processo do auth é o **único** que carrega a
  chave privada de assinatura e os segredos de provedor (Resend/Twilio/OAuth), guardados em
  Rails credentials / secret manager.
- **IMPORTANTE: Devise sozinho NÃO é o IAM da casa.** Devise/Sorcery cobrem login básico
  (sessão, `has_secure_password`, reset por email), mas **não** entregam por desenho
  ID≠email + ≥2 fatores + ReBAC multi-tenant + sessão longa/logout irreversível. Podem servir
  de **base só dentro do app de auth separado** (`<projeto>_auth_rb`), **completando o piso**
  desta skill — nunca como "o IAM já está pronto porque tem Devise".
- **O app principal (e todo cliente) delega ao auth por OIDC/OAuth2.1 + PKCE:** redireciona
  pra `auth.<domain>`, recebe tokens de volta. O serviço Ruby de auth é o **IdP da casa**
  (padrão self-hosted, consumido por N apps) — expõe os endpoints padrão
  (`/authorize`, `/token`, `/introspect`, `/revoke`, `/.well-known/openid-configuration`,
  `/.well-known/jwks.json`).
- **Segredos e chave de assinatura de token vivem SÓ no serviço de auth Ruby**; consumidores
  (middleware Rack dos outros serviços) validam por **JWKS público** cacheado — nunca
  guardam a chave privada. A assinatura JWT (Ed25519/EdDSA ou RS256, via gem `jwt`) só
  acontece dentro de `<projeto>_auth_rb`; a rotação de chave é publicada via `kid` no JWKS.

## 2. Modelo de identidade

- **ID interno imutável e opaco** (ULID/UUIDv7, `SecureRandom.uuid_v7`) é o `sub`. **Email e
  telefone NUNCA são ID** — são *identificadores* ligados ao usuário, cada um com estado de
  verificação. Em Ruby, o modelo `User` tem `id` opaco (PK não sequencial); `Email`/`Phone`
  são `has_many` (entidades filhas) com `verified_at`, nunca a chave.
- **Identificadores 1..N por usuário** (emails, telefones, identidades SSO, passkeys, apps,
  chaves FIDO2). **Ter mais de um email é incentivado** (resiliência a brick de provedor).
- **Identificador só vale verificado** — não loga nem recupera sem verificação.
- **SSO nunca é ponto único de falha:** cadastro via SSO **força ≥1 fator de recuperação
  local** (email de recuperação + códigos de backup), pra provedor banido ≠ conta perdida.
- **Account-linking explícito:** SSO chegando com email já verificado em outra conta →
  linkar vs. bloquear **com confirmação** (anti-takeover). Nunca linkar por email não
  verificado.
- **Nudge de email secundário (anti-brick):** com só 1 email, a UI **sugere adicionar um
  secundário**. **Detecta o provedor** do atual (gmail / hotmail-outlook / yahoo /
  próprio-corporativo) e **recomenda que o secundário seja preferencialmente de OUTRO
  provedor**. Ao lado, um **"i" com tooltip no hover**: *"Um segundo email, de preferência
  em outro provedor, garante que você não perca o acesso caso perca acesso a este email."*
  Sugestão, não obrigação. (Regra do domínio, exposta pelo auth ao `authfront`.)

## 3. Fatores e níveis de garantia (AAL — NIST 800-63B)

Classificar a **força** de cada fator permite "email sempre disponível" sem abrir mão de
segurança: operação sensível exige fator forte; email/SMS servem de fallback.

| Tier | Fatores | Uso |
|---|---|---|
| **Alto (phishing-resistant)** | **Passkey/WebAuthn (núcleo)**, chave FIDO2, push aprovado no app | Ops sensíveis: trocar fator, admin, cross-tenant, billing, recuperação |
| **Médio** | TOTP (app autenticador), senha + posse | Login + 2º fator |
| **Baixo (fallback)** | **Email OTP (Resend)**, **SMS/voz (Twilio)** | **É o 2º fator baseline da conta** (senha+OTP = 2FA); sempre disponível; **não** autoriza ação sensível sozinho (aí exige step-up forte) |

- **Email OTP (Resend/Postmark) ligado por padrão, inclusive em HML** — só o operador desliga.
  **Ligado ≠ entregando pra fora:** em `production` o `delivery_method` é o real; **fora dele a
  entrega é sinkada** (`:test`/`letter_opener`/Mailpit) e o **interceptor** recusa destinatário
  fora do domínio de teste (§3.1) — o OTP é exercido ponta-a-ponta lendo
  `ActionMailer::Base.deliveries` ou a caixa do Mailpit, nunca uma caixa real.
- **Twilio por padrão** para verificação de telefone e 2FA por SMS/voz.
- **Provedores plugáveis como POROs/adaptadores:** o core depende de uma interface (duck type),
  não do SDK — cada provedor é um Plain Old Ruby Object com contrato estável:
  ```ruby
  # Contrato (duck type): responde a send_otp(to:, code:)
  class ResendEmailProvider   # def send_otp(to:, code:) ... end
  class TwilioSmsProvider     # def send_otp(to:, code:) ... end
  class ApnsPushProvider      # def request_approval(device_token:, challenge:) ... end
  ```
  `EmailProvider` (Resend default), `SmsProvider` (Twilio default), `PushProvider` são
  **trocáveis por config/wiring** (injeção via initializer), sem tocar no core (adaptadores
  na borda, DDD).
- **WebAuthn/passkey** por gem madura (**`webauthn`**); **TOTP** por **`rotp`** (+ `rqrcode`
  pro QR). Assinatura/JWT por gem **`jwt`** auditada.
- **Senha por padrão, opcional por escolha:** o usuário **cria senha no cadastro** (padrão
  cultural; **argon2id** via gem **`argon2`** + verificação contra base de vazadas/HIBP
  k-anonymity), mas o **seletor de modos de autenticação permite marcá-la como opcional** e
  viver de passkey/OTP/app. `has_secure_password` só com cost explícito quando usar bcrypt.
- **Passkey/WebAuthn é núcleo** (não roadmap): já é "2 fatores num" (dispositivo +
  biometria), phishing-resistant.
- **2FA por desenho desde o cadastro — senha + Email OTP JÁ é 2FA (fraco, porém válido):**
  a conta **nasce com dois fatores obrigatórios** (senha + código no email verificado,
  always-on) e **já é segura para o baseline**. **VETADO** tratar senha+OTP como "sem 2FA" e
  **barrar o login** até enrolar um fator forte — é o **círculo infinito**. Em Ruby: o
  `before_action` de sessão **libera o acesso baseline** com a sessão de AAL médio (senha+OTP);
  **não** barra todas as rotas exigindo AAL alto.
- **Fator forte é INCENTIVADO + just-in-time, nunca muro pré-login:** app OTP / passkey / chave
  são **nudge** e **exigidos só na operação sensível** (o PEP checa o AAL mínimo **por rota
  sensível** e dispara **step-up** ali) e **escalados sob risco** (§9). Enrolar um fator forte
  usa o Email OTP como verificação (Y≠X, §4): sem deadlock. A ausência de fator forte **degrada
  o que é sensível** (`403 step_up_required`), não **bloqueia o baseline**.

### 3.1 Efeito externo fora de `production` — sink por default e guard como INTERCEPTOR

O OTP é o fluxo que **mais amplifica envio**: um `db:seed` de 5.000 contas dispara 5.000 e-mails.
Por isso **ligado ≠ entregando pra fora**. Em `production` o `delivery_method` é o real (sem
credencial, o boot **falha** — não cai em sink silencioso); em **qualquer outro ambiente** a
entrega é sinkada e o guard **recusa destinatário fora do domínio de teste**. O porquê
(bounce/complaint em massa queima IP e domínio e **derruba o OTP de produção**, com semanas de
warm-up e utilidade zero) e as 4 camadas estão na normativa: `schematize-engineering` →
`references/efeitos-externos.md`. Aqui vai só o **recorte Ruby/Rails**.

**Por que INTERCEPTOR e não um wrapper no mailer:** `ActionMailer::Base.register_interceptor`
roda `delivering_email(message)` **antes de toda entrega, de TODO mailer** — inclusive os que
alguém escrever depois, os de gem, os disparados por `deliver_later` dentro do worker e os de
`ActionMailer::MessageDelivery` chamados de rake task. Um guard dentro de `UserMailer` protege
`UserMailer`; o interceptor protege **o processo**. Levantar exceção no interceptor **aborta a
entrega** — nada é passado ao `delivery_method`.

#### Camada 1 — `delivery_method` por ambiente (`config/environments/*.rb`)

```ruby
# config/environments/test.rb
# :test não abre socket nenhum — acumula em ActionMailer::Base.deliveries.
config.action_mailer.delivery_method       = :test
config.action_mailer.perform_deliveries    = true  # o interceptor PRECISA rodar
config.action_mailer.raise_delivery_errors = true  # erro de entrega REPROVA o teste
config.action_mailer.default_url_options   = { host: "test.acme.com" }

# config/environments/development.rb
# letter_opener abre no browser; Mailpit é a caixa de verdade (API HTTP pro teste ler).
config.action_mailer.delivery_method       = :letter_opener
# Alternativa Mailpit/MailHog local:
# config.action_mailer.delivery_method = :smtp
# config.action_mailer.smtp_settings   = { address: "127.0.0.1", port: 1025 }
config.action_mailer.raise_delivery_errors = true
config.action_mailer.perform_caching       = false

# config/environments/staging.rb  (hml — ambiente Rails próprio, NÃO "production com flag")
config.action_mailer.delivery_method = :smtp
config.action_mailer.smtp_settings   = { address: ENV.fetch("MAILPIT_HOST"), port: 1025 }

# config/environments/production.rb
config.action_mailer.delivery_method = :smtp
config.action_mailer.smtp_settings   = {
  address:              ENV.fetch("SMTP_ADDRESS"),   # fetch SEM default: sem credencial o
  port:                 ENV.fetch("SMTP_PORT", 587), # boot falha, em vez de sinkar em silêncio
  user_name:            ENV.fetch("SMTP_USER"),
  password:             ENV.fetch("SMTP_PASSWORD"),  # chave de PRD, que não existe em outro env
  authentication:       :plain,
  enable_starttls_auto: true,
}
config.action_mailer.raise_delivery_errors = true
```

#### Camada 2 — o guard (interceptor), em `lib/` e registrado sempre

```ruby
# lib/external_recipient_guard.rb
# frozen_string_literal: true

# Interceptor de saída de e-mail: roda ANTES de toda entrega, em TODO mailer do
# app. Fora de production, destinatário que não seja do domínio de teste em rota
# nula é ERRO — nunca warning, nunca no-op silencioso.
#
# Mora em lib/ (não em app/) de propósito: é `require`d no initializer e NÃO é
# recarregável — instância registrada no boot não pode apontar pra classe velha
# depois de um reload do Zeitwerk.
class ExternalRecipientGuard
  # Erros de PROGRAMAÇÃO: aparecem no teste, no CI e no log — não são erro do usuário.
  ExternalRecipientBlocked = Class.new(StandardError)
  RunCapExceeded           = Class.new(StandardError)

  # 1º = o `test.<domain>` da casa (null MX RFC 7505 + SPF `-all` + DMARC p=reject).
  # Demais = TLDs reservados da RFC 2606/6761, que não resolvem em lugar nenhum.
  DEFAULT_TEST_DOMAINS = %w[test.acme.com test invalid example example.com].freeze

  DEFAULT_MAX_PER_RUN = 50

  # @param env [String] Rails.env (só a string exata "production" entrega de verdade)
  # @param test_domains [Array<String>] domínios aceitos fora de production
  # @param max_per_run [Integer] teto de mensagens por processo (MAIL_MAX_PER_RUN)
  def initialize(env: Rails.env.to_s,
                 test_domains: DEFAULT_TEST_DOMAINS,
                 max_per_run: Integer(ENV.fetch("MAIL_MAX_PER_RUN", DEFAULT_MAX_PER_RUN), exception: false) || DEFAULT_MAX_PER_RUN)
    @env          = env
    @test_domains = test_domains.map(&:downcase).freeze
    @max_per_run  = max_per_run
    @sent         = 0
    @mutex        = Mutex.new # Puma/Sidekiq entregam em threads: o contador é compartilhado
  end

  attr_reader :max_per_run

  # Contrato do ActionMailer: chamado para CADA mensagem, antes da entrega.
  # Levantar aqui aborta a entrega — nada chega ao delivery_method.
  # @param message [Mail::Message]
  # @return [void]
  def delivering_email(message)
    return if production? # produção entrega de verdade

    blocked = recipients(message).reject { |address| test_recipient?(address) }
    if blocked.any?
      raise ExternalRecipientBlocked, <<~MSG.squish
        bloqueado: destinatário externo #{blocked.join(', ')} com Rails.env=#{@env}.
        Use <papel>+<run-id>-<n>@#{@test_domains.first} ou registre o ADR de exceção
        (allowlist <=5 + cap + janela + subdomínio de envio separado). Nada foi enviado.
      MSG
    end

    count = @mutex.synchronize { @sent += 1 }
    return if count <= @max_per_run

    raise RunCapExceeded, <<~MSG.squish
      cap de envio estourado (#{count} > MAIL_MAX_PER_RUN=#{@max_per_run}, Rails.env=#{@env}):
      laço/seed disparando em massa. Abortado — nenhum e-mail a mais foi entregue.
    MSG
  end

  # Zera o contador entre execuções (usado pelo suite de teste, nunca em runtime).
  # @return [void]
  def reset!
    @mutex.synchronize { @sent = 0 }
  end

  private

  # Fail-closed: qualquer env que não seja exatamente "production" é não-produção.
  def production? = @env == "production"

  def recipients(message)
    Array(message.to) + Array(message.cc) + Array(message.bcc)
  end

  # Match por DOMÍNIO (não por substring): "notgmail.test.acme.com.br" não passa.
  def test_recipient?(address)
    domain = address.to_s.downcase.split("@").last.to_s
    return false if domain.empty?

    @test_domains.any? { |allowed| domain == allowed || domain.end_with?(".#{allowed}") }
  end
end
```

```ruby
# config/initializers/external_recipient_guard.rb
# frozen_string_literal: true
require "external_recipient_guard"

# Registrado SEM condição de ambiente, de propósito: o no-op de produção mora
# DENTRO do guard. Registro condicional erra fechado no lado errado — se o env
# for detectado torto, a versão condicional fica SEM guard nenhum.
# Registro direto (não em `to_prepare`): a classe vem de lib/ e não é recarregável,
# então registrar uma vez no boot evita interceptor duplicado a cada reload do dev.
ActionMailer::Base.register_interceptor(ExternalRecipientGuard.new)
```

#### Camada 3 — o teste que vê o vermelho

```ruby
# spec/mailers/external_recipient_guard_spec.rb
require "rails_helper"

RSpec.describe ExternalRecipientGuard do
  before { ActionMailer::Base.deliveries.clear }

  it "recusa destinatário externo fora de production e NÃO entrega nada" do
    expect {
      UserMailer.otp(to: "alguem@gmail.com", code: "123456").deliver_now
    }.to raise_error(described_class::ExternalRecipientBlocked, /destinatário externo/)

    expect(ActionMailer::Base.deliveries).to be_empty
  end

  it "entrega no domínio de teste em rota nula" do
    UserMailer.otp(to: "login+run42-1@test.acme.com", code: "123456").deliver_now

    expect(ActionMailer::Base.deliveries.size).to eq(1)
    expect(ActionMailer::Base.deliveries.last.to).to eq(["login+run42-1@test.acme.com"])
  end

  it "aborta ao estourar o cap por execução" do
    guard = described_class.new(env: "test", max_per_run: 2)
    message = ->(n) { Mail.new(to: "carga+run7-#{n}@test.acme.com") }

    guard.delivering_email(message.call(1))
    guard.delivering_email(message.call(2))

    expect { guard.delivering_email(message.call(3)) }
      .to raise_error(described_class::RunCapExceeded, /MAIL_MAX_PER_RUN/)
  end

  it "não trata env desconhecido como produção (fail-closed)" do
    guard = described_class.new(env: "prod-like")

    expect { guard.delivering_email(Mail.new(to: "alguem@gmail.com")) }
      .to raise_error(described_class::ExternalRecipientBlocked)
  end
end
```

```ruby
# Minitest — test/mailers/external_recipient_guard_test.rb
class ExternalRecipientGuardTest < ActionMailer::TestCase
  test "recusa destinatário externo" do
    assert_raises(ExternalRecipientGuard::ExternalRecipientBlocked) do
      UserMailer.otp(to: "alguem@gmail.com", code: "123456").deliver_now
    end
    assert_empty ActionMailer::Base.deliveries
  end
end
```

**O que isso trava, e o que continua sendo sua responsabilidade:**

| Regra | Onde está no código |
|---|---|
| Guard pega **todo** mailer, inclusive os futuros | `register_interceptor` no initializer, não um wrapper por mailer |
| Sink por default fora de prd | `delivery_method` `:test`/`:letter_opener`/Mailpit SMTP em `config/environments/*.rb` |
| Fail-closed | `production?` só é verdade em `Rails.env == "production"`; registro **incondicional** |
| Cap por execução + abort | `@max_per_run` + `RunCapExceeded` (contador com `Mutex` — Puma/Sidekiq são multi-thread) |
| Recusa é **erro**, não warning | `raise` dentro de `delivering_email` aborta a entrega |
| Destinatário sintético em rota nula | `DEFAULT_TEST_DOMAINS`, com match por **domínio**, não substring |

- **`deliver_later` não é bypass:** o interceptor roda no processo que entrega (o worker Sidekiq),
  desde que o initializer esteja carregado lá — o que é automático no boot do Rails. Mas o
  **contador é por processo**: web e worker têm caps independentes (é o esperado; o cap é freio
  de laço, não cota global).
- **Fixture, seed, FactoryBot e `simulated`:** todo endereço é
  `<papel>+<run-id>-<n>@test.<domain>` — em FactoryBot, `sequence(:email) { |n| "user+#{RUN_ID}-#{n}@test.acme.com" }`.
  **VETADO** `@gmail.com`/`@hotmail.com`, domínio de terceiro, e-mail de pessoa real (inclusive o
  seu) e o domínio de produção em `spec/`, `test/`, `db/seeds.rb` ou qualquer rake task.
- **SMS/push/webhook/PSP seguem o MESMO desenho:** o adaptador (PORO) recebe o guard por injeção
  no initializer, com o mesmo cap (SMS custa por unidade — o cap importa mais, não menos) e chave
  **sandbox** fora de prd. Twilio tem `test credentials` e magic numbers; use-os.
- **WebMock/VCR ligados no `spec_helper`** (`WebMock.disable_net_connect!(allow_localhost: true)`)
  são a 4ª rede: nenhuma chamada HTTP de teste sai da máquina, nem por SDK que ignore o mailer.

## 4. Fluxos

**Onboarding:** cita um email → **verifica** → **cria senha** (ou já passkey/app) → **pronto:
2FA baseline (senha + Email OTP) e acesso baseline pleno**. Só **depois**, já dentro, o sistema
**sugere** (nudge, não obriga) reforçar: 2º email de backup + fator forte. **Nunca se barra o
acesso por não ter fator forte** — ele é pedido *just-in-time* na 1ª ação sensível (step-up)
ou sob risco (§9).

**Login:** (1) sem app de 2FA ativo → **OTP por email** (mesmo sem nada habilitado); (2)
com app → **pergunta app ou email**; (3) com vários fatores (passkey/telefone/app) →
**lista todos e o usuário escolhe** qual usar. App = push-approval ou TOTP.

**Gestão de fator — invariante único:**
> **Para mutar o fator X, apresente um fator Y ≠ X, no maior AAL disponível.**
- Desativar/trocar **app** → verifica por **email** (ou outro ≠ app).
- Trocar/adicionar **email complementar** → exige o **app**.
- Add/remover **chave ou telefone** → mesmo princípio, **lista qual usar**.
- Toda mudança **notifica todos os canais verificados**; remover o **último fator forte** =
  **ação com atraso cancelável** (janela pra abortar se for ataque) — implementada como job
  agendado (Sidekiq/GoodJob) no auth, cancelável até disparar.

**Recuperação:** múltiplos caminhos independentes (vários emails, códigos de backup
offline, telefone). **Força ≥ login** (2 fatores ou processo com atraso + revisão),
rate-limit agressivo, tudo auditado. **Reset nunca é bypass de 1 fator.**

## 5. Multi-tenant + RBAC/ABAC — motor ReBAC (estilo Zanzibar)

- **Identidade global, autorização por tenant:** um usuário (1 identidade) pertence a **N
  tenants** via **membership**, com papéis **diferentes por tenant**.
- **Motor de relação (ReBAC), ex. OpenFGA/SpiceDB** — hand-rolar authz é onde vazam
  privilégios. O serviço Ruby **não implementa o motor**: fala com o OpenFGA/SpiceDB via
  client gRPC/HTTP (gem oficial). Autorização em **tuplas** `(objeto, relação, usuário/userset)`:
  - `tenant:acme#member@user:01H…`
  - `role:acme/finance-approver#assignee@user:01H…`
  - `invoice:987#parent@tenant:acme` (recurso parenteado ao tenant)
  - permissão computada por *relation rewrite* (member do tenant **E** assignee de papel
    que concede a permissão).
  - **Escrita de tuplas** (conceder/revogar papel, criar membership) passa por um caso de
    uso do auth que chama o `Write` do motor **transacionalmente com o outbox** — nunca um
    dual-write solto.
- **RBAC granular:** permissão = **`recurso:ação`** (`invoice:approve`, `user:invite`);
  papéis-padrão (owner/admin/member/viewer) **+ papéis 100% customizados e granulares por
  tenant** (viram relations/usersets). Deve ser possível criar cargos extremamente
  granulares e atribuí-los.
- **ABAC por cima:** condições sobre atributos (usuário/recurso/contexto — hora, IP, risco)
  via **conditional/contextual tuples** (ex.: aprova invoice < 10k do próprio setor).
- **PDP/PEP separados:** PDP = **Check API** do motor; **PEP = middleware Rack / `before_action`
  de controller** em cada serviço (`require_permission("invoice:approve")`), que extrai
  `sub`+tenant do token e chama `check(obj, rel, subject)`. **Deny-by-default** (erro/timeout
  do PDP = nega), enforcement **server-side**, **todo endpoint mapeia 1 permissão**.
- **Token fino:** o JWT carrega `sub`/tenant/sessão/AAL — **sem** a lista de permissões
  (evita authz stale em token longo); a decisão é consultada no PDP e cacheada com TTL
  curto (in-memory/Redis) invalidado por evento de mudança de papel.
- **Toda decisão de authz é logada** (quem / o quê / allow-deny / política) — auditoria +
  rotina de testes.

## 6. Sessão, multi-dispositivo e logout

- **Multi-dispositivo de 1ª classe:** N sessões simultâneas por usuário, cada uma atada a
  um **dispositivo** (fingerprint + rótulo amigável "Chrome no Windows", IP/geo, último
  uso). Nenhuma sessão derruba a outra. O **session store** (Postgres + Redis) guarda uma
  linha por sessão com `refresh_family_id`, `device_id`, `jti` corrente.
- **View de dispositivos/sessões:** lista os ativos e **permite remover um** (revoga a
  sessão daquele device), além de **"sair de todos"**.
- **Sessão longa por padrão (fim do "15 min e é chutado"):** o access token continua curto
  (ex.: 15 min) **mas com refresh silencioso** — para o usuário, a sessão **persiste 7 dias
  por padrão**. No login, **pergunta se o dispositivo é confiável**; se sim, **90 dias**.
  Ops sensíveis ainda pedem **step-up fresco** em AAL alto — sessão longa não enfraquece.
- **Refresh rotativo com detecção de reuso** (reusou um refresh já rotacionado → revoga a
  **família** inteira). O refresh token é opaco (CSPRNG, `SecureRandom`), hasheado no store.
- **Botão "Sair" bem visível → kill IRREVERSÍVEL da sessão:** não basta apagar o cookie —
  o handler de logout do auth Ruby **revoga o refresh token (e a família), apaga o registro
  de sessão server-side, joga o `jti` na denylist (Redis/DB) até expirar e desassocia o
  push token do device**. Depois do logout, aquela sessão é irrecuperável: nem replay, nem
  refresh, nem "voltar o cookie" reativa. O middleware de validação de access token
  **consulta a denylist de `jti`** em toda request (cache curto).
- Cookies **`HttpOnly` + `Secure` + `SameSite`**; token nunca em `localStorage`.

## 7. Migração de auth legado — PRIORIDADE 0

Existe auth no padrão antigo → **portar pra este IAM é prioridade máxima** (segurança
inegociável; pode gastar o que precisar). Estratégia **strangler-fig** (casa com
schematize-node quando o legado é Node): dual-run, **re-hash preguiçoso** no login
(verifica no hash legado, regrava em **argon2id**), mapeia registros legados → modelo novo
(dedupe de emails, cunha IDs internos ULID/UUIDv7), **ativa o Email OTP always-on como 2º
fator baseline** (a conta migrada já entra em 2FA sem muro) e **incentiva enrolar fator forte**
(step-up para sensível), **revoga sessões legadas** e **nunca confia na authz legada**
(re-deriva as tuplas ReBAC). O auth migrado nasce já como **aplicação Ruby separada** (§1).

## 8. Rotina agressiva de testes (detalhe na schematize-pentest)

Suíte adversarial **contínua** (CI + agendada, fixtures multi-tenant, saída
machine-readable, **gate que trava** em qualquer vazamento):
- **Cross-tenant (BOLA/IDOR):** token do tenant B → IDs do tenant A = 403/404; fuzz de IDs.
- **Priv-esc (BFLA):** papel baixo → ação de papel alto (horizontal e vertical).
- **Matriz persona × endpoint** exaustiva.
- **Abuso de fluxo:** bypass de 2FA, reset pulando 2FA, brute-force/rate-limit de OTP,
  replay de token, reuso de refresh, JWT `alg=none`/kid trocado, session fixation,
  adulteração de asserção SSO, IDOR na gestão de identificadores, bypass de step-up,
  mass-assignment de papel, **logout que não invalidou de verdade** (sessão recuperável).

## 9. Autenticação adaptativa por risco (robusta) + transversais

A resposta ao login **varia com o risco calculado** (não é fixa) — é o que torna a conta
difícil de tomar sem chatear o legítimo:
- **Log de sessões/tentativas:** cada tentativa e sessão gravam IP/ASN+reputação, device
  fingerprint, geo, UA, horário, resultado e **score de risco** — na view de sessões (§6) e em
  audit log imutável. É o insumo do score.
- **Score por tentativa:** IP suspeito/novo (Tor/proxy/ASN de abuso), device novo, geovelocidade
  impossível, velocity/brute, hit de honeypot. Baixo = fluxo normal; alto = escala.
- **Escalonamento por risco (2FA→3FA):** sob risco, exige um **fator a mais na ordem de força**
  — **senha → código por email → app OTP/chave**. Acertar senha+email não basta em contexto
  suspeito. Mesmo motor do step-up (§3), disparado pelo **contexto**, não só pela ação.
- **Negação deceptiva / tarpit (falso negativo sob risco):** em contexto suspeito, mesmo com
  **senha correta** o serviço pode responder **genérico `invalid_credentials` uma vez** enquanto
  **computa server-side que a credencial estava certa** e marca que a **próxima** tentativa
  correta **passa** (já com os fatores escalados). Seguro porque: **resposta e tempo IDÊNTICOS**
  ao erro real (sem oráculo — use `ActiveSupport::SecurityUtils.secure_compare` em tempo
  constante e o mesmo path de resposta); estado "próxima passa" **curto e escopado** (conta+IP+
  device, TTL curto, expira sozinho, nunca vira lockout do legítimo); **soma-se** ao 3FA, não
  substitui; tudo logado.
- **Honeypot:** contas/campos/rotas isca; qualquer interação = sinal forte de hostil → score
  alto, tarpit/deceção, alerta. Nunca serve tráfego real.
- **Anti-automação sempre:** rate-limit (`rack-attack`) + **backoff exponencial** e **lockout
  progressivo por conta+IP**; OTP curto, single-use, `jti` na denylist. Barra o abuso sem
  derrubar o serviço.
- **Notifica o usuário:** login novo/suspeito, novo device, mudança de credencial → aviso nos
  canais verificados, com "não fui eu" (revoga + força reforço).

### Transversais (sempre)
- **Audit log imutável** de toda decisão authn/authz e mudança de credencial — alimenta a
  forense e os testes (liga com a observabilidade LGTM+ da casa; spans OpenTelemetry no
  fluxo de login/authz com `trace_id`).
- **Padrões:** OIDC/OAuth2.1 + PKCE; WebAuthn/FIDO2; AALs NIST 800-63B; SCIM (roadmap
  enterprise); FAPI2 se fintech.

## Roadmap de fases
- **F0** Núcleo de identidade (ID imutável, N identificadores, verificação, Resend/Twilio
  plugáveis por PORO/adaptador, email OTP always-on) — já como **aplicação Ruby separada**
  em `auth.<domain>`.
- **F1** 2FA baseline por desenho (senha + Email OTP, sem muro pré-login) + fluxos (TOTP/
  push, **passkey**, escolha de método, invariante de troca, **nudge** de fator forte, step-up
  just-in-time, **risk engine adaptativo**: score, 2FA→3FA, negação deceptiva/tarpit, honeypot).
- **F2** Multi-tenant + **ReBAC** (membership, papéis granulares, PDP/PEP em middleware Rack/
  `before_action`, deny-default, token fino, audit).
- **F3** Recuperação resiliente (múltiplos caminhos, força ≥ login, SSO com recuperação
  local, atraso cancelável, nudge de email secundário).
- **F4** App de 1ª classe + multi-dispositivo (OIDC/PKCE nativo, push-approval, view de
  remover dispositivos, sessão 7d/90d confiável, logout irreversível).
- **F5** Migração de legado (prioridade 0 quando aplicável).
- **F6** Rotina agressiva de testes (cross-tenant, priv-esc, abuso de fluxo — CI + agendada).
- **Roadmap+** chave FIDO2 dedicada, SCIM, FAPI2, trusted contacts.

## Checklist (entra na Definition of Done quando o projeto tem auth)
- [ ] Auth é **app separada** — serviço Ruby `<projeto>_auth_rb` + front próprios em `auth.<domain>`, isolados — não monolith/engine. **Devise sozinho não conta como IAM da casa.**
- [ ] **ID interno imutável** (ULID/UUIDv7, `SecureRandom.uuid_v7`); email/telefone não são ID; múltiplos emails suportados.
- [ ] **2FA baseline por desenho** (senha + Email OTP = 2FA desde o cadastro); fator forte é **nudge + just-in-time (step-up)**, **NUNCA muro pré-login** (o `before_action` libera o baseline, exige AAL alto só por rota sensível); passkey (gem `webauthn`) no núcleo; email OTP always-on; Twilio; providers como POROs/adaptadores.
- [ ] **Efeito externo só sai em `production` (§3.1):** `delivery_method` **sinkado** fora de prd (`:test` no test, `letter_opener`/Mailpit no dev, Mailpit no staging), **`ExternalRecipientGuard` registrado por `ActionMailer::Base.register_interceptor`** (pega todo mailer, inclusive os futuros) levantando `ExternalRecipientBlocked` quando o destinatário está fora do domínio de teste e `Rails.env != "production"`, **cap por execução** (`MAIL_MAX_PER_RUN`, default 50) com `RunCapExceeded`, **fail-closed** (só a string `"production"` entrega) e chave real **ausente** fora de prd (`ENV.fetch` sem default: o boot falha). Prova: spec que espera o `raise_error` e `expect(ActionMailer::Base.deliveries).to be_empty` (vermelho visto) + `dig MX test.<domain>` = null MX. (`schematize-engineering` → `references/efeitos-externos.md`)
- [ ] **Risk engine adaptativo:** log de sessões/tentativas + score (IP/device/geo/velocity/honeypot); **2FA→3FA** sob risco; **negação deceptiva/tarpit** (falso negativo, resposta idêntica ao erro real em tempo constante via `secure_compare`, "próxima passa" curta/escopada); notifica login suspeito.
- [ ] Invariante de troca de fator (Y≠X, maior AAL); recuperação ≥ login; SSO com recuperação local; senha em **argon2id** (gem `argon2`)+HIBP.
- [ ] **Multi-tenant + RBAC/ABAC** (ReBAC OpenFGA/SpiceDB), deny-default, PDP=Check API / PEP=middleware Rack/`before_action`, enforcement server-side, token fino.
- [ ] Multi-dispositivo + view de remover; **sessão 7d/90d**; **logout irreversível** (revoga refresh+família, `jti` na denylist, não só cookie).
- [ ] JWKS público (assinatura só no auth, gem `jwt`); audit log de authn/authz; risk engine/rate-limit (`rack-attack`); migração de legado tratada como prioridade 0.
- [ ] Rotina agressiva de testes cross-tenant/priv-esc no CI (schematize-pentest).
