# Segurança, Auth, Multi-tenancy, LGPD e Frontend

> Parte da skill **schematize-ruby**. As referências cruzadas (§N) apontam para seções do corpo completo — todas presentes no conjunto de references desta skill.

## Índice
- 13. Segurança
- 14. Autenticação e Autorização
- 15. Multi-tenancy
- 32. LGPD e Dados Pessoais
- 38. Frontend / NextJS / SPA — Regras Específicas

---

## 13. Segurança

**MUST — pipeline (fail on high/critical)**
- Dependabot ou Renovate (Gemfile + gems transitivas).
- SAST: **`brakeman`** (Rails) travando o build em achado alto; Semgrep/CodeQL para regras extras.
- SCA: **`bundler-audit`** (CVE de gem, base do rubysec) + verificação de licença — liga com `cadeia-suprimentos.md`.
- Container scan (Trivy / Grype).
- Secret scan (Gitleaks) — inclui `config/master.key`, `config/credentials/*.key`, `.env`.
- Commit signing (GPG / SSH / Sigstore).
- SBOM no build (CycloneDX via `cyclonedx-ruby`).

**MUST — runtime**
- Containers não-root, read-only filesystem.
- Imagem `ruby:<versão>-slim` ou distroless/chainguard quando viável; nunca `latest`.
- Multi-stage build (bundle no estágio de build; runtime só com o necessário).
- Healthcheck na imagem.
- `.ruby-version` pinado e dentro do suporte (EOL casa com `stack-versoes.md`).

**MUST — segredos**
- Nunca em código, nunca no `Gemfile`, nunca em env file commitado.
- Storage: **Rails credentials** (`config/credentials.yml.enc` com a **master key FORA do repo**) ou Vault / AWS/GCP Secret Manager / ENV injetada. `.env` **nunca** commitado.
- Rotação documentada; `master.key`/`RAILS_MASTER_KEY` só no ambiente, nunca no git.

**Licenças permitidas:** MIT, Apache 2.0, BSD, MPL 2.0, ISC.
**Bloqueadas sem ADR:** GPL/AGPL, SSPL, proprietárias.

### 13.4 Segredos no Cliente / Frontend / NextJS

**VETADO — sem exceção, sem ADR**

- **Qualquer segredo no bundle que vai pro browser.** API key privada, secret de JWT, senha de banco, service-role key (Supabase/Firebase admin), token de provedor de pagamento, chave de terceiro — **nada** disso entra em código que o cliente baixa. O navegador não guarda segredo. Ponto.
- Prefixar segredo com `NEXT_PUBLIC_` (ou equivalente `VITE_`, `REACT_APP_`, `PUBLIC_`). Esse prefixo **expõe a variável publicamente por definição** — usar só para valor que pode estar num outdoor.
- Chamar API de terceiro com chave secreta **direto do browser**. Toda chamada que usa segredo passa por um **BFF / route handler / server action / controller Rails** server-side (§38).
- Guardar token de sessão em `localStorage`/`sessionStorage`. Sessão vai em **cookie `HttpOnly` + `Secure` + `SameSite`** (§38, a `schematize-qa` (smoke e matriz simulated, `references/categorias.md` secoes 5 e 10) hardening).
- Confiar em validação de auth/role feita no client (`if (user.isAdmin)` no React) como controle de acesso. Isso é UX, não segurança — a decisão é **sempre server-side** (§15, §37).

> "Coloca a senha no ERB/no NextJS pra funcionar" não é uma solução, é um vazamento agendado. Se o cliente pode ver, o atacante já viu.

---

---

## 14. Autenticação e Autorização

- OAuth2 / OIDC.
- JWT assinado (RS256 ou EdDSA; **nunca HS256** em fluxo público). Gem `jwt` com `verify: true` e algoritmo **explícito**.
- Validação **completa** do JWT em toda request: assinatura, `exp`, `nbf`, `aud`, `iss` e `alg` contra **allowlist** — **VETADO `alg=none`** e VETADO aceitar o algoritmo que vem no header. Decodar o payload e confiar (`JWT.decode(t, nil, false)`) é VETADO (§37).
- Refresh token rotativo com detecção de reuso.
- RBAC com permissões granulares; ABAC quando necessário. Detalhe do IAM da casa em `iam.md`.
- Hash de senha: **bcrypt cost ≥ 12** (gem `bcrypt` + `has_secure_password` com cost explícito) ou **argon2id** (gem `argon2`). MD5/SHA1/sem-salt/plaintext são VETADOS (§37).
- Tokens, ids de sessão e códigos de reset gerados por **CSPRNG** (`SecureRandom.hex`/`.uuid_v7`) — **nunca `rand`/`Random`** (§37).
- **CSRF ligado** (`protect_from_forgery`); APIs tratadas explicitamente (token/header, não desligar cegamente).

---

---

## 15. Multi-tenancy

**MUST — quando aplicável (SaaS, plataformas)**
- Isolamento de tenant explícito em todas as camadas.
- `tenant_id` propagado em contexto, logs e traces.
- Autorização validada **server-side** em toda operação. **Nunca** confiar em `tenant_id` (ou role, ou user_id) vindo de `params`/body sem verificação contra o token. Derivar sempre do token verificado, server-side.
- Queries com `tenant_id` no `WHERE` (`where(tenant_id: current_tenant.id)`), sempre. `default_scope` de tenant ajuda mas **não é authz sozinho**. Considerar Row Level Security no Postgres.

**SHOULD**
- Testes de cross-tenant leak em CI.
- Métricas e logs particionáveis por tenant.

---

---

## 32. LGPD e Dados Pessoais

**MUST**
- Classificação de dados: público, interno, confidencial, pessoal, pessoal sensível.
- PII nunca em logs (ver §16.1; usar `filter_parameters` do Rails para senha/token/CPF).
- Política de retenção documentada por tipo de dado.
- Processo para exercício de direitos (acesso, correção, eliminação, portabilidade).
- Criptografia em trânsito (TLS 1.2+) e em repouso para dados pessoais (Active Record Encryption para campos sensíveis).
- DPIA para tratamentos de alto risco.
- PII **nunca** em query string / URL (acaba em log, histórico, referer) — §37.

---

---

## 38. Frontend / NextJS / SPA — Regras Específicas

**MUST**
- **Fronteira clara client/server.** Tudo que toca segredo, banco, ou terceiro com credencial roda **server-side** (controller/service Rails, route handler, server action, BFF, server component que não serializa segredo pra props).
- Sessão em **cookie `HttpOnly` + `Secure` + `SameSite=Lax|Strict`**. Token de auth **nunca** em `localStorage`/`sessionStorage` (XSS lê tudo lá).
- Variáveis públicas (`NEXT_PUBLIC_*` etc.) contêm **apenas** dado não-sensível (URL de API pública, id de analytics público). Tratar esse prefixo como "vai pro outdoor".
- Validação de input no client é **UX**; a validação que importa é a do servidor — **strong parameters** (`params.require(...).permit(...)`) e/ou `dry-validation` (§12). Autorização idem é server-side (§15).
- CSP, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy` e `frame-ancestors` configurados (a `schematize-qa` (smoke e matriz simulated, `references/categorias.md` secoes 5 e 10) hardening).
- Chamada a API de terceiro com chave secreta passa por proxy server-side. O browser nunca segura a chave.

**VETADO**
- Service-role key / admin SDK (Supabase, Firebase, etc.) no código do client.
- `permit!` / `params.to_unsafe_h` em input de usuário (mass-assignment — §37).
- `dangerouslySetInnerHTML` / `render inline:` / ERB com input não sanitizado, e `html_safe`/`raw` em conteúdo de usuário (XSS).
- Confiar em `redirect`/`next` param sem allowlist (open redirect — a `schematize-qa` (smoke e matriz simulated, `references/categorias.md` secoes 5 e 10)).

> "Bota a senha no ERB" não existe como solução. Existe como CVE. O front pede ao servidor; o servidor guarda o segredo.

---

---
