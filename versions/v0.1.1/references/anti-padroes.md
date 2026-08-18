# Filosofia, Aplicação Universal e Anti-Padrões Vetados

> Parte da skill **schematize-ruby**. As referências cruzadas (§N) apontam para seções do corpo completo — todas presentes no conjunto de references desta skill.

## Índice
- 0. Como Ler
- 0.1. Aplicação Universal — Este Arquivo é Contexto Máximo
- 1. Filosofia
- 37. Anti-Padrões Vetados — "Macaquices" que Terminam Rápido e Quebram em Produção

---

## 0. Como Ler

- **MUST / Obrigatório** — regra. Desvio bloqueia merge ou exige ADR.
- **SHOULD / Recomendado** — padrão. Desvio precisa de justificativa no PR.
- **MAY / Opcional** — sugestão.
- **VETADO / Proibido** — não existe "atalho". Não se faz, não se cogita, não se "resolve depois". Burlar é incidente, não decisão técnica.

Quando este documento conflitar com a realidade do problema, **registre um ADR** explicando o desvio. Padrão sem exceção vira dogma; dogma vira dívida. **Mas itens marcados VETADO não têm ADR de exceção** — são pisos de segurança e integridade, não preferências.

Versões concretas de Ruby/Rails (e janela de EOL) ficam em **`references/stack-versoes.md`**, atualizado independentemente deste corpo. Os pisos agnósticos vêm de `schematize-engineering`; teste de segurança é a `schematize-pentest`.

---

---

## 0.1. Aplicação Universal — Este Arquivo é Contexto Máximo

**MUST**

- Este documento é **anexado a TODO prompt / sessão / tarefa** de engenharia (humana ou assistida por IA). É **contexto pinado**, não referência opcional. Se a tarefa toca código, infra, dados, deploy ou design de sistema, este arquivo está em contexto. Sem exceção.
- Nenhuma resposta, PR, geração de código ou decisão arquitetural é válida se ignorar este documento. "Não estava no contexto" não é desculpa — **garantir o anexo é responsabilidade de quem abre a tarefa**, e a ausência dele é, por si só, motivo para parar e recarregar o contexto antes de produzir qualquer coisa.
- Em caso de conflito entre uma instrução pontual ("faz rápido", "ignora o teste", "depois a gente arruma") e este documento, **este documento vence**. Pressa não revoga regra.
- Assistentes de IA operam sob as mesmas regras dos humanos (§34) **e** sob as proibições explícitas da §37. A dinamicidade do Ruby (metaprogramação, monkey patch, duck typing) **não** é licença pra pular teste ou tipo — ela **aumenta** a dependência do teste, porque o compilador não segura a barra por você.

> Um padrão que não está no contexto na hora da decisão é um padrão que não existe. Por isso ele é pinado, não linkado.

---

---

## 1. Filosofia

Prioridades, em ordem de desempate:

1. Clareza > esperteza (em Ruby: legibilidade > "one-liner esperto" com metaprogramação)
2. Simplicidade > abstração antecipada
3. Manutenibilidade > velocidade pontual
4. Observabilidade > debugging manual
5. Segurança por padrão > segurança como camada final
6. Evolução incremental > big-bang
7. **Registro do que foi decidido > memória de quem decidiu** (ver §28)

**Princípios:** Clean Code, SOLID, KISS, DRY (com bom senso — duplicação acidental ≠ duplicação semântica). Ruby convida à esperteza; a casa prefere o método explícito e nomeado ao truque de `method_missing`.

**Regra suprema:** se algo aumenta acoplamento, reduz observabilidade, cria dependência desnecessária, **vaza segredo, mistura contexto, ou pula registro** ou adiciona complexidade sem benefício claro, **provavelmente está errado**.

---

---

## 37. Anti-Padrões Vetados — "Macaquices" que Terminam Rápido e Quebram em Produção

> Atalhos que parecem entregar mais rápido e na real entregam vulnerabilidade, vazamento ou dívida. **Todos VETADOS** — não admitem ADR de exceção, são pisos. Aparecem em diff humano ou de IA → o PR para. Cada item traz o **veto** e o **caminho certo**.

### Segredos e exposição

1. **Segredo no código ou no `Gemfile`.** API key, secret de JWT, senha de banco, service-role key, token de pagamento hardcoded, ou credencial em URL de gem no `Gemfile`.
   → Segredo **só server-side**, em **Rails credentials** (`config/credentials.yml.enc` + master key fora do git) ou **ENV**. O cliente/browser nunca guarda segredo (§13.4, §38).

2. **PII / token / senha em query string ou URL.** Acaba em log do Puma/Nginx, `production.log`, histórico do browser, header `Referer`.
   → Vai em body ou header apropriado, nunca na URL. Filtre em `config.filter_parameters` (§32, §16.1).

3. **`credentials`/`.env`/master key real commitado**, ou segredo hardcoded "temporário".
   → Rails credentials criptografado + `.env.example` sem valores; `RAILS_MASTER_KEY` no secret manager. Gitleaks no pipeline (§13).

### Injeção e execução

4. **SQL por concatenação/interpolação de string** — `where("email = '#{params[:email]}'")`, `find_by_sql("... #{input}")`, `order(params[:sort])` cru.
   → **Placeholders/hash sempre**: `where("email = ?", email)` ou `where(email: email)`. Nunca interpolar input em `where`/`find_by_sql`/`order`/`pluck` (§10). `brakeman` pega isso.

5. **`eval`, `instance_eval`, `send`/`public_send` sobre input, `system`/`` `backticks` ``/`%x`/`Kernel#open` com string de input.**
   → Nunca. Args separados (`system("cmd", arg1, arg2)`), allowlist de comandos/métodos. `Kernel#open` com `|` executa shell — use `File.open`/`URI.open` conscientemente.

6. **Desserialização insegura de dado não-confiável** — `YAML.load`, `Marshal.load`, `Oj` em modo objeto sobre payload externo (RCE clássico em Ruby/Rails).
   → `YAML.safe_load` (com allowlist de classes), `JSON.parse`, nunca `Marshal.load` de fonte não-confiável (§10).

7. **Desabilitar verificação TLS** (`OpenSSL::SSL::VERIFY_NONE`, `verify_mode: OpenSSL::SSL::VERIFY_NONE`, `verify: false` no Faraday/HTTParty) pra "funcionar logo".
   → Cert válido. mTLS interno. Se o cert está errado, conserta o cert.

### Auth e autorização

8. **Auth/authz só no client** (`if current_user.admin?` decidido só na view/JS).
   → Toda decisão de acesso é **server-side** (§15), no controller/policy. Front é UX, não controle.

9. **Confiar em `tenant_id` / `role` / `user_id` vindos de `params`/body/header do cliente** sem validar contra o token/sessão.
   → Derivar sempre do token verificado / `current_user` server-side (§15). `params[:tenant_id]` é hostil por padrão.

10. **JWT decodado sem validar** assinatura, `exp`, `aud`, `iss`, e `alg` contra allowlist (aceitar `alg: none`, ou HS256 com pubkey RS256).
    → Validação completa em toda request; fixe `algorithm:` explicitamente no `JWT.decode` (§14).

11. **Hash de senha fraco** — MD5, SHA1, `Digest`, sem salt, ou plaintext.
    → `has_secure_password` (bcrypt cost ≥ 12) ou **argon2id** (§14). Nunca rolar hash à mão.

12. **`rand`, `Random`, `SecureRandom` ausente pra token, id de sessão, código de reset, nonce** — usar `rand(10**6)` pra OTP, `SecureRandom.random_number` de baixa entropia.
    → **`SecureRandom`** (`.hex`, `.uuid`, `.base58`) com entropia adequada; CSPRNG sempre (§14).

### Mass assignment, CORS e superfície

13. **Mass assignment sem strong params** — `Model.new(params[:user])` / `update(params.to_unsafe_h)`, deixando passar `admin`, `tenant_id`, `created_at`, `password_digest`.
    → **Strong params** com `permit(...)` allowlist explícita por endpoint; nunca `permit!`.

14. **`Access-Control-Allow-Origin: *` em rota autenticada** (pior com `credentials: true`) no `rack-cors`.
    → Allowlist explícita de origens no `config/initializers/cors.rb` (§22.3 hardening).

15. **Endpoint de debug/admin (Sidekiq Web UI, `/rails/info`, console web) sem auth, ou bind em `0.0.0.0`** expondo porta interna.
    → Bind restrito, auth obrigatória, montar Sidekiq/admin atrás de `authenticate` no routes; `web-console` só em dev (§22.3).

### Erros, tipos e qualidade

16. **`rescue` que engole erro** — `rescue nil`, `rescue => e; end`, `rescue StandardError; end` sem tratar/logar, `rescue Exception`.
    → Tratar, logar com contexto e `trace_id`, propagar ou degradar de forma consciente. Nunca `rescue nil`. Resgatar `StandardError` específico, jamais `Exception`.

17. **Silenciar RuboCop/Sorbet pra calar o linter** — `# rubocop:disable` amplo, `T.unsafe`, `T.untyped` pra fugir do tipo, `# rubocop:disable Security/*`.
    → Tipar/tratar de verdade. Inline-disable de cop **de segurança** (`Security/*`) ou de `brakeman` é VETADO sem ADR.

18. **Logar request/response inteiro, headers ou body cru "pra debugar".**
    → Logar campos específicos, mascarados via `filter_parameters`. Nunca PII/token/senha (§16.1).

### Testes e cobertura

19. **Pular/comentar teste pra passar o CI** — `xit`, `skip`, `pending`, `focus` esquecido, comentar o `expect`.
    → Conserta o código, não silencia o teste. A dinamicidade do Ruby **exige** o teste (RSpec/Minitest + FactoryBot); ele é a rede que o compilador não te dá.

20. **Baixar o threshold do SimpleCov ou editar o gate** pra o número fechar.
    → Cobertura é contrato (§22). Sobe escrevendo teste, não mexendo na régua.

21. **Mockar o próprio sistema sob teste** retornando sucesso fixo, ou `allow(...).to receive` no objeto testado, dando "verde" falso.
    → Testar comportamento real; mock/stub só nas bordas externas (gateway, HTTP, relógio).

### Dados e migrations

22. **Migration sem `down` (ou não-reversível sem `up`/`down` explícito), ou destrutiva sem backup** (`drop_table`/`remove_column`/`change_column` que perde dado).
    → Reversível (`change` reversível, ou `up`+`down`), testada com `db:rollback` antes do merge (§10). ActiveRecord adverte migração irreversível — respeite.

23. **Cache de resposta autenticada sem chave por usuário/tenant** — `Rails.cache`/fragment cache global, um user recebe dado do outro.
    → Chave de cache sempre segmentada por usuário e tenant (§11, §15).

### Operação e entrega

24. **Container root, `chmod 777`, `--privileged`, filesystem RW** "pra funcionar".
    → Não-root, read-only, least-privilege (§13).

25. **Gem nova sem verificar** nome (typosquatting), manutenção, licença — ou **`Gemfile.lock` não commitado / `bundle install` sem lockfile**, versão frouxa (`gem "x"` sem pin).
    → `Gemfile.lock` commitado é **piso**; `bundle install --frozen`/`--deployment` no CI/deploy. Pin, checar nome/manutenção/licença, **bundler-audit** + **brakeman** no pipeline (§13, §34).

26. **Retry infinito / sem backoff/jitter** — Sidekiq com retry sem limite, loop de `net/http` sem teto — DoS no próprio sistema ou no terceiro.
    → Limite explícito + backoff exponencial + jitter (`sidekiq_options retry:`) + DLQ (§9, §18).

27. **`Idempotency-Key` aceito mas ignorado** (header existe, lógica não), ou job Sidekiq não-idempotente que duplica efeito no retry.
    → Implementar a deduplicação de fato; job idempotente (§12).

28. **Dual-write** — gravar no banco e publicar no broker/enfileirar Sidekiq no mesmo fluxo, sem outbox (job enfileirado antes do commit da transação = fantasma).
    → Transactional Outbox; enfileirar **após** o commit (`after_commit`) ou via outbox (§9, Anexo B).

29. **Pular o archive/MD "pra ir mais rápido"** (§28).
    → Archive é parte da entrega. Sempre gerado. Tarefa sem archive não está pronta (§35).

30. **Merge direto na `main` / force push em branch protegida / pular o PR e o review.**
    → Trunk-based com PR, CI verde, CODEOWNERS (§24).

31. **Desligar rate limit, validação de payload (strong params/`rack-attack`), ou security scan "temporariamente".**
    → "Temporário" vira permanente. Não se desliga piso de segurança (§12, §13).

32. **Criar serviço backend novo FORA DO ROL sancionado, ou sem ADR de fit.** Ruby **É** do rol (permitido por **fit + ADR**, `references/arquitetura.md` §3) — o veto aqui é escolher linguagem por gosto sem ADR, ou reabrir o que está em saída: **Node como serviço backend** e **PHP** **não recebem serviço novo** (são legado que migra por funcionalidade do módulo — ~30% extrai, ~50% termina; PHP migra sumariamente).
    → Serviço novo escolhe uma do rol (Go/Rust/Elixir/C#/Zig/Ruby) com ADR justificando o fit; Ruby quando o fit manda (protótipo, script, DX de produto Rails). Frontend segue Node via `schematize-web`. Nova linguagem fora do rol = ADR de exceção (§3, `schematize-engineering`/`references/linguagens.md`).

33. **Ruby EOL em produção**, ou não pinar `.ruby-version` (`rbenv`/`asdf`) — rodar versão sem patch de segurança.
    → `.ruby-version` pinado numa versão **não-EOL**; janela de suporte em `references/stack-versoes.md`. Atualização de runtime é manutenção, não opcional.

34. **Serviço que não sobe / crasha porque outro serviço está fora** (acoplamento de runtime, crash em cascata) — "a `api` não sobe sem o `core`".
    → Cada serviço é entidade à parte: sobe e opera sozinho; dependente ausente = degradação graciosa, nunca crash (§2, §18).

35. **Falha ao notificar/chamar outro serviço que derruba o chamador ou perde o dado** sem persistir, alertar e retomar.
    → Store-and-forward: persiste o intento (outbox/Sidekiq/Redis/DB), loga com `trace_id`, alerta (Grafana), retoma com retry+backoff+jitter → DLQ + escala (§18).

36. **Criar arquivos ou repos fora da pasta do projeto** (começar largando arquivos no root e depois **subir de diretório** — `cd ..`, `../` — pra criar repos irmãos fora; ou espalhar em `~`, `~/Documents`, `~/Downloads`, `/tmp`, Área de Trabalho).
    → Aplicação nova = **pasta dentro do workspace atual** (`./<projeto>_<contexto>/`). O agente não sai da pasta do projeto (ler ou escrever) sem o usuário pedir (§2).

37. **Editar código direto no servidor** (hml/prd), ou **subir mudança direto pra hml/prd** pulando `dev local → teste local → GitHub`.
    → Servidor é **imutável por edição manual**; recebe só artefato promovido do git. Hotfix segue o mesmo fluxo, acelerado (`references/ops.md` §1).

38. **Operar o servidor por fora do `<projeto>_ops`** — `ssh` + comando ad-hoc, `rails console`/`rails runner` na prod na mão pra corrigir dado, editar arquivo no servidor, `docker`/`kubectl`/`systemctl` solto.
    → **100%** de install/update/config/correção passa por comando do ops. Correção de dado é migration/script versionado, não `console` na produção. Não tem comando? **cria no ops** (`references/ops.md` §2).

39. **Instalar/subir o sistema em série** ("um serviço de cada vez", 20 min).
    → Instalação **paralela por padrão** = `nproc` (`references/ops.md` §3).

40. **Serializar a instalação pra "funcionar"**, mascarando que um serviço depende de outro pra subir.
    → Erro que só ocorre em paralelo = **serviços não independentes** (fere piso 34). O ops **expõe** a colisão; corrigir a independência é **prioridade máxima**. Nunca esconder com serialização (`references/ops.md` §6).

41. **Redeploy que faz patch in-place / não parte do seed** (estado acumulado, drift entre implantações — `bundle install` sobre uma app velha, asset stale).
    → Todo redeploy é **destrutivo na app**: apaga a anterior e recria um clone zerado a partir de `/<app>/.env` (`references/ops.md` §2). Idempotente e reprodutível.

42. **Metaprogramação mágica sem doc + teste** — `method_missing`/`define_method`/`send` dinâmico que esconde o comportamento do índice de microfunções e do stack trace; **monkey patch** de classe de terceiro/core sem refinement e sem ADR.
    → Método explícito e nomeado por padrão. Metaprogramação necessária = documentada em YARD (métodos gerados) + coberta por teste; monkey patch via `refinements` com escopo e ADR (`references/padroes-codigo.md` §3).

43. **Config/segredo de serviço fora do seed global**, ou repos do sistema espalhados fora de `/<app>/`.
    → `/<app>/.env` é a **fonte única** de config; o ops clona os repos dentro de `/<app>/` (`references/ops.md` §2).

44. **Apagar dados persistentes num redeploy** ("destrutivo" incluindo banco/volumes), ou `ops reset` de dados em prd.
    → Destrutivo é a **aplicação, nunca os dados**: banco/volumes/uploads preservados (migration reversível); apagar dado é `ops reset` **gated a dev/hml** (`references/ops.md` §2).

45. **Dois serviços no mesmo user Linux, serviço Puma/Sidekiq rodando como `root`, ou criar user/unit/permissão à mão.**
    → **Um user + systemd unit hardened por serviço**, provisionado **pelo ops** (`references/ops.md` §3). Blast radius mínimo.

### IAM (identidade e autorização)

46. **Auth apensado ao escopo principal como monolith** (login/2FA/authz dentro do Rails principal, num `app/models/user` que faz tudo, sem serviço/front próprios).
    → Auth é **app separada** em `auth.<domain>`: serviço `<projeto>_auth_rb` + `<projeto>_authfront`, isolados; apps delegam por OIDC/PKCE e validam por JWKS público (`references/iam.md` §1).

47. **Email/telefone como ID de usuário** (`user_id = email`, FK por email, login que assume 1 email), ou 2FA/recuperação com 1 fator só (reset por 1 email que pula o 2FA).
    → **ID interno imutável** (ULID/UUIDv7) como `sub`; email/telefone são identificadores N e verificáveis; **≥2 fatores sempre** (passkey/WebAuthn no núcleo); **recuperação ≥ força do login**; senha em **argon2id**+HIBP (`references/iam.md` §2–§4).

48. **Autorização hand-rolled / no cliente / permissão embutida em token longo** — `if current_user.admin?` espalhado pelos controllers, `before_action` frouxo, checagem só na view, sem multi-tenant, papéis não-granulares.
    → **RBAC/ABAC granular por motor ReBAC** (OpenFGA/SpiceDB via client), **deny-default**, PDP=Check API / PEP=**policy/middleware Ruby** (Pundit/Action Policy como camada fina sobre o motor), **enforcement server-side**, token fino, decisão auditada (`references/iam.md` §5).

49. **Logout que só apaga o cookie** (sessão recuperável por refresh/replay), ou sessão curta que chuta o usuário toda hora sem refresh silencioso.
    → **Logout irreversível** (revoga refresh+família, apaga sessão server-side, `jti` na denylist consultada em toda request); **sessão 7d/90d** com refresh rotativo silencioso e multi-dispositivo (`references/iam.md` §6).

> Regra de bolso: se a justificativa começa com "só pra funcionar", "depois eu arrumo", ou "é mais rápido assim" e o resultado mexe em segredo, auth, dado, registro, **ou toca o servidor por fora do fluxo/ops** — **provavelmente é uma macaquice desta lista. Para e faz certo.**

---

---
