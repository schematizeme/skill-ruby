# Segurança, Auth, Multi-tenancy, LGPD e Frontend


> **PONTEIRO, não cópia.** A normativa deste tema é da base: **`schematize-engineering`** →
> `references/seguranca.md`. Leia lá primeiro; aqui fica **só o que muda em Ruby**.
>
> **Onde este arquivo divergir da base, a BASE MANDA** (`SKILL.md` §"Precedência e herança").
> Em 2026-08-21 os blocos idênticos à base foram **podados mecanicamente** (`tools/podar-clone.mjs`),
> e é por isso que a numeração **salta**: o número é o da base, e o item ausente aqui é o que **não
> muda nesta linguagem** — procure-o lá.

## 13. Segurança

- Dependabot ou Renovate (Gemfile + gems transitivas).

- SAST: **`brakeman`** (Rails) travando o build em achado alto; Semgrep/CodeQL para regras extras.

- SCA: **`bundler-audit`** (CVE de gem, base do rubysec) + verificação de licença — liga com `cadeia-suprimentos.md`.

- Container scan (Trivy / Grype).

- Secret scan (Gitleaks) — inclui `config/master.key`, `config/credentials/*.key`, `.env`.

- Commit signing (GPG / SSH / Sigstore).

- SBOM no build (CycloneDX via `cyclonedx-ruby`).

- Imagem `ruby:<versão>-slim` ou distroless/chainguard quando viável; nunca `latest`.

- Multi-stage build (bundle no estágio de build; runtime só com o necessário).

- `.ruby-version` pinado e dentro do suporte (EOL casa com `stack-versoes.md`).

- Nunca em código, nunca no `Gemfile`, nunca em env file commitado.

- Storage: **Rails credentials** (`config/credentials.yml.enc` com a **master key FORA do repo**) ou Vault / AWS/GCP Secret Manager / ENV injetada. `.env` **nunca** commitado.

- Rotação documentada; `master.key`/`RAILS_MASTER_KEY` só no ambiente, nunca no git.

### 13.4 Segredos no Cliente / Frontend / NextJS

- Chamar API de terceiro com chave secreta **direto do browser**. Toda chamada que usa segredo passa por um **BFF / route handler / server action / controller Rails** server-side (§38).

- Guardar token de sessão em `localStorage`/`sessionStorage`. Sessão vai em **cookie `HttpOnly` + `Secure` + `SameSite`** (§38, a `schematize-qa` (`references/categorias.md` §§5 e 10) hardening).

> "Coloca a senha no ERB/no NextJS pra funcionar" não é uma solução, é um vazamento agendado. Se o cliente pode ver, o atacante já viu.

## 14. Autenticação e Autorização

- JWT assinado (RS256 ou EdDSA; **nunca HS256** em fluxo público). Gem `jwt` com `verify: true` e algoritmo **explícito**.

- Validação **completa** do JWT em toda request: assinatura, `exp`, `nbf`, `aud`, `iss` e `alg` contra **allowlist** — **VETADO `alg=none`** e VETADO aceitar o algoritmo que vem no header. Decodar o payload e confiar (`JWT.decode(t, nil, false)`) é VETADO (§37).

- RBAC com permissões granulares; ABAC quando necessário. Detalhe do IAM da casa em `iam.md`.

- Hash de senha: **argon2id** (gem `argon2`) para senha nova — parâmetros mínimos no `iam.md` da base
  (`schematize-engineering` -> `references/iam.md` §2.1); **bcrypt cost ≥ 12** só como legado a migrar.
  MD5/SHA1/sem-salt/plaintext são VETADOS (§37).

- **Login com `authenticate_by`, nunca `find_by` + `authenticate`.** No Rails 7.1+,
  `User.authenticate_by(email: ..., password: ...)` existe por um motivo específico: escrito ao pé
  da letra, o login "óbvio" (`User.find_by(email:)&.authenticate(password)`) **não faz o trabalho de
  hash quando o e-mail não existe** — e responde **muito mais rápido** nesse caso. Isso é
  **enumeração de usuário por timing**, medível de fora com uma amostra pequena. O
  `authenticate_by` gasta o mesmo tempo nos dois caminhos (faz um hash descartável quando não
  acha o registro). O par disso é a resposta genérica: *"e-mail ou senha inválidos"*, igual nos dois
  casos.

- Tokens, ids de sessão e códigos de reset gerados por **CSPRNG** (`SecureRandom.hex`/`.uuid_v7`) — **nunca `rand`/`Random`** (§37).

- **CSRF ligado** (`protect_from_forgery`); APIs tratadas explicitamente (token/header, não desligar cegamente).

## 15. Multi-tenancy

- Autorização validada **server-side** em toda operação. **Nunca** confiar em `tenant_id` (ou role, ou user_id) vindo de `params`/body sem verificação contra o token. Derivar sempre do token verificado, server-side.

- Queries com `tenant_id` no `WHERE` (`where(tenant_id: current_tenant.id)`), sempre. `default_scope` de tenant ajuda mas **não é authz sozinho**. Considerar Row Level Security no Postgres.

## 32. LGPD e Dados Pessoais

- PII nunca em logs (ver §16.1; usar `filter_parameters` do Rails para senha/token/CPF).

- Criptografia em trânsito (TLS 1.2+) e em repouso para dados pessoais (Active Record Encryption para campos sensíveis).

## 38. Frontend / NextJS / SPA — Regras Específicas

- **Fronteira clara client/server.** Tudo que toca segredo, banco, ou terceiro com credencial roda **server-side** (controller/service Rails, route handler, server action, BFF, server component que não serializa segredo pra props).

- Validação de input no client é **UX**; a validação que importa é a do servidor — **strong parameters** (`params.require(...).permit(...)`) e/ou `dry-validation` (§12). Autorização idem é server-side (§15).

- CSP, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy` e `frame-ancestors` configurados (hardening; ver a `schematize-qa`, `references/categorias.md` §§5 e 10).

- `permit!` / `params.to_unsafe_h` em input de usuário (mass-assignment — §37).

- `dangerouslySetInnerHTML` / `render inline:` / ERB com input não sanitizado, e `html_safe`/`raw` em conteúdo de usuário (XSS).

- Confiar em `redirect`/`next` param sem allowlist (open redirect — a `schematize-qa` (`references/categorias.md` §§5 e 10)).

> "Bota a senha no ERB" não existe como solução. Existe como CVE. O front pede ao servidor; o servidor guarda o segredo.
