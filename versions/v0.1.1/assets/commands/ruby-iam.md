---
description: schematize-ruby — força/audita/scaffolda o IAM da casa (identidade≠email, ≥2 fatores, ReBAC multi-tenant, sessão longa/logout irreversível) como app Ruby separada em auth.<domain>, ou atualiza um auth existente
argument-hint: "[bootstrap | audit | migrate]"
---

Governe identidade e autorização pelo padrão IAM da casa (`references/iam.md`), no recorte
**backend/Ruby**. Plan-first: **audita, mostra o plano, pede aprovação, então executa.** Use
este comando para **forçar só a parte de IAM** num projeto (novo ou existente) ou
**atualizar/portar um auth legado**.

## 0. Modo
- `audit` — varre o projeto e reporta o gap contra o piso IAM (checklist §iam).
- `bootstrap` — scaffolda o IAM do zero como **app Ruby separada**.
- `migrate` — porta um auth legado pro IAM (**prioridade 0**, strangler-fig).

## 1. Topologia primeiro (inegociável)
Confirme/scaffolde que o auth é **aplicação SEPARADA** (`references/iam.md` §1):
- Serviço **Ruby** próprio `<projeto>_auth_rb` (Rails/Sinatra/Roda) + front próprio
  `<projeto>_authfront`, servidos em **`auth.<domain>`** — **VETADO** apensar como
  engine/monolith no serviço principal.
- **Devise/Sorcery sozinho NÃO é o IAM da casa** — cobrem login básico, mas não entregam
  ID≠email + ≥2 fatores + ReBAC multi-tenant + sessão longa/logout irreversível por
  desenho. Se usados, é **dentro** do app de auth separado e completando o piso.
- Repo/deploy/**user Linux + systemd isolados** por conta própria (casa com `ops.md` §3;
  app Puma e Sidekiq como units à parte).
- App principal e clientes **delegam por OIDC/OAuth2.1 + PKCE**; a **chave de assinatura só
  no auth Ruby**, consumidores validam por **JWKS público** (middleware cacheando o JWKS).

## 2. Identidade
- **ID interno imutável** (ULID/UUIDv7 via `SecureRandom.uuid_v7`/gem `ulid`) como `sub`;
  **email/telefone nunca são ID**; **N emails** por usuário (entidades filhas com
  `verified_at`, nunca a PK). Identificador só vale **verificado**.
- **SSO com recuperação local forçada**; account-linking explícito (anti-takeover).
- **Nudge de email secundário:** detecta provedor e recomenda outro + tooltip "i".

## 3. Fatores (≥2 sempre)
- **Passkey/WebAuthn no núcleo** (gem `webauthn`); TOTP (gem `rotp`)/push; **email OTP
  always-on inclusive HML** (provider tipo Resend/Postmark; só operador desliga); **Twilio**
  p/ telefone; providers **plugáveis como POROs/adaptadores** na borda.
- **Senha por padrão** (**argon2id** via gem `argon2` + HIBP), **opcional no seletor**.
- **Senha + Email OTP já é 2FA baseline:** o **PEP libera o acesso baseline** e exige AAL
  alto **só por rota sensível** (step-up just-in-time, `403 step_up_required`), **nunca barra
  o login** por falta de fator forte. Fator forte é **nudge + just-in-time**.
- Invariante de troca: **mutar fator X exige fator Y≠X no maior AAL**; notificar canais;
  remover último fator forte = **atraso cancelável** (job agendado). Recuperação **≥ login**.

## 4. Autorização (multi-tenant, ReBAC)
- **Identidade global, papéis por tenant** (membership). Motor **ReBAC (OpenFGA/SpiceDB)** —
  o auth Ruby **fala com o motor via client**, não implementa authz na mão; tuplas
  `(objeto, relação, usuário)`; **RBAC granular** (`recurso:ação`, papéis customizados) +
  **ABAC** (conditional tuples). **PDP = Check API / PEP = middleware Rack ou `before_action`,
  deny-default, enforcement server-side, token fino, decisão auditada.** Escrita de tupla via
  outbox (nunca dual-write solto).

## 5. Sessão / logout
- **Multi-dispositivo** + **view de remover dispositivos** + "sair de todos" (session store
  Postgres+Redis, 1 linha por sessão com `refresh_family_id`/`device_id`/`jti`).
- **Sessão 7 dias por padrão; pergunta se confiável → 90 dias** (access token curto com
  refresh silencioso rotativo — nada de "15 min e é chutado"). Step-up fresco em ops
  sensível. Refresh opaco (`SecureRandom`), reuso → revoga a **família**.
- **Botão Sair visível → kill IRREVERSÍVEL:** revoga refresh + família, apaga sessão
  server-side, `jti` na **denylist** (Redis/DB, consultada em toda request), desassocia push
  token. Nada recria a sessão.

## 6. Testes (dispare o gate do pentest)
Rode/priorize a rotina agressiva (`schematize-pentest`): **cross-tenant (BOLA/IDOR),
priv-esc (BFLA), abuso de fluxo (bypass 2FA/reset/step-up, replay, refresh reuse, JWT
alg=none/kid, logout que não invalidou)** — gate que trava em vazamento. Ver `/pentest-authz`.

## 7. Migração de legado (modo `migrate`, prioridade 0)
Strangler-fig: dual-run, **re-hash preguiçoso** (→argon2id) no login, mapeia registros →
modelo novo (dedupe emails, cunha IDs ULID/UUIDv7), **força 2º fator no 1º login**, **revoga
sessões legadas**, **re-deriva authz** em tuplas ReBAC (nunca confia na antiga). O auth
migrado nasce app Ruby separada. (Legado Node casa com `schematize-node`.)

## 8. Saída
Grave o plano/relatório em `<projeto>_archive/` (§28): topologia (app separada?), gaps do
checklist IAM (`references/iam.md`), plano por fase (F0–F6) e — se `migrate` — o mapa
legado→novo e a ordem de corte. Confirme: auth é app Ruby à parte? Devise não é o IAM sozinho?
identidade≠email? ≥2 fatores? ReBAC multi-tenant deny-default? sessão longa + logout
irreversível? testes cross-tenant no CI?
