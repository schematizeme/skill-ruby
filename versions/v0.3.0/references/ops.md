<!-- cross-skill: references/orquestracao.md -> schematize-engineering -->

# Operação pelo ops — ambientes, instalação e correção (control plane)


> **PONTEIRO, não cópia.** A normativa deste tema é da base: **`schematize-engineering`** →
> `references/ops.md`. Leia lá primeiro; aqui fica **só o que muda em Ruby**.
>
> **Onde este arquivo divergir da base, a BASE MANDA** (`SKILL.md` §"Precedência e herança").
> Em 2026-08-21 os blocos idênticos à base foram **podados mecanicamente** (`tools/podar-clone.mjs`),
> e é por isso que a numeração **salta**: o número é o da base, e o item ausente aqui é o que **não
> muda nesta linguagem** — procure-o lá.

> O **`<projeto>_ops`** é a **interface única** de operação do sistema. Invariantes desta reference, todos **INEGOCIÁVEIS**: (1) nada chega ao servidor sem passar pelo **fluxo de promoção** (dev local → teste local → GitHub → hml → prd); (2) o ops provisiona num **workspace por aplicação** (`/<app>/`) e todo **redeploy é destrutivo na aplicação, semeado pelo `/<app>/.env` global** — mas **nunca destrói dados**; (3) cada serviço roda **isolado por usuário** (user Linux + systemd hardened); (4) **100%** de instalação/atualização/correção/config passa por uma **ferramenta do ops** — nunca à mão; (5) instalar é **paralelo por padrão** (= `nproc`), e falha no paralelo = **bug de independência** (prioridade máxima). Contexto do ops: `arquitetura.md` §2. Deploy/ambientes: `operacao.md` §21. Test kit: `testes.md` a `schematize-qa` (test kit, `references/execucao.md` secao 2).

## 1. Ambientes e o fluxo de promoção (nada direto no servidor)

- **Nada pula etapa.** Nunca vai direto pra hml ou prd. hml só recebe o que está no GitHub e passou no teste local (`bundle exec rspec` verde); prd só recebe o que foi homologado em hml.

- **VETADO editar código direto no servidor (hml/prd).** O servidor é **imutável por edição manual** — recebe apenas **artefato promovido do git** (mesma imagem/commit SHA, §21). "Editei o `.rb` direto no prd pra resolver rápido" é o anti-padrão que esta reference existe pra matar.

## 2. Layout no servidor, seed global e redeploy destrutivo

- **`/<app>/` é a raiz da aplicação no servidor.** O ops **cria** essa pasta e **clona os repos dentro** dela: `/<app>/<app>_<contexto>` (ex.: `/loja/loja_api_rb`, `/loja/loja_worker_rb`, `/loja/loja_ops`). Nada de repo espalhado pelo host.

- **`/<app>/.env` é o SEEDER GLOBAL — a fonte única.** Toda a configuração da aplicação inteira parte desse arquivo. É a **única** fonte de verdade de config em runtime; **nenhum serviço tem config à parte** fora do seed (nada de `.env` por repo ou `credentials.yml.enc` divergente). (Segredo real via secret manager referenciado pelo seed — segredo nunca versionado no repo.)

- **Redeploy é DESTRUTIVO na aplicação, a partir do seed.** Todo (re)deploy: **apaga a implantação anterior** e **recria um clone novo, zerado**, com `bundle install --deployment --frozen` e config renderizada **exclusivamente** a partir de `/<app>/.env`. Sem patch in-place, sem gem instalada à mão, sem drift. O estado da app é 100% derivado do seed → **idempotente e reprodutível**: `ops redeploy` sempre entrega o mesmo resultado, do zero.

- **"Destrutivo" é a APLICAÇÃO, nunca os DADOS.** O que é destruído e recriado: o clone dos repos, `vendor/bundle`/build, config renderizada, os processos Puma/Sidekiq. O que é **preservado**: **dados persistentes** (banco, volumes, uploads, filas Redis/Sidekiq), que só mudam por **migration reversível** (com `down` + backup, §21). Apagar dados é comando **separado e gated** (`ops reset`), **só em dev/hml**, com confirmação explícita — **nunca** em prd.

## 3. Isolamento por usuário — um user + systemd por serviço

- **Um usuário Linux dedicado por serviço/repo.** `loja_api_rb` (Puma) roda como user `loja_api_rb`; `loja_worker_rb` (Sidekiq) como user `loja_worker_rb`. **Nunca** dois serviços no mesmo user; **nunca** `root`. A pasta de cada repo pertence ao user do serviço (isolamento a nível de usuário no filesystem). O worker Sidekiq é **unit à parte** do web Puma — users, units e pastas distintos.

- **Um systemd unit isolado por serviço**, rodando como o user dedicado, com **hardening**: `User=<svc>`, `NoNewPrivileges=yes`, `ProtectSystem=strict`, `ProtectHome=yes`, `PrivateTmp=yes`, `ReadWritePaths=` só o necessário, `CapabilityBoundingSet=` mínimo. Puma e Sidekiq têm cada um seu unit.

- **Automatizado pelo ops, SEMPRE.** Criar o user, ajustar permissão da pasta, gerar o systemd unit hardened (Puma e Sidekiq) e fazer o wiring é **feito pelo ops** (`ops install`/`ops redeploy`) — **nunca à mão**. É parte do provisionamento, não um passo manual esquecível.

## 4. O ops é a interface única (100% das operações)

- **Proibido** `ssh servidor` + comando ad-hoc, editar arquivo no servidor, `bundle`/`rails`/`systemctl`/`sidekiqctl` na mão, script solto. Se **não existe** comando de ops pra aquilo, **cria o comando no ops** — não faz por fora. O que não está no ops não aconteceu (e não é reproduzível).

- **Superfície mínima** (idempotentes, com `--help` e saída machine-readable): `bootstrap` (cria `/<app>/` e clona os repos) · `install`/`up` · `redeploy` (destrutivo, do seed §2) · `update` · `config` (do `/<app>/.env`) · `migrate` (reversível, `rails db:migrate`/`db:rollback`) · `health`/`doctor` · `rollback` · `logs`/`troubleshoot` · `reset` (destrói dados — gated, dev/hml) · `test` (ver a `schematize-qa` (test kit, `references/execucao.md` secao 2)).

## 5. Instalação SEMPRE paralela (= nº de cores)

- O ops instala/sobe/recria **módulos e microserviços em paralelo**, grau = **`nproc`** (default; `--jobs N`/`OPS_JOBS` sobrepõe). O `bundle install` de cada repo roda em paralelo entre repos independentes.

- **Serialização só onde há dependência real e declarada** (ex.: `rails db:migrate` antes do serviço que consome o schema) — o **mínimo**, explícito no grafo de instalação, nunca "serializa tudo por via das dúvidas".

## 6. Independência é invariante — falha no paralelo = bug (PRIORIDADE MÁXIMA)

- **O ops NÃO contorna serializando "pra funcionar".** Serializar pra mascarar a dependência é **proibido** — esconde o defeito. O ops **expõe** a colisão (relatório: o que disputou o quê, quem esperou quem) e a correção real é feita: **cada serviço sobe sozinho**, sem exigir outro no boot, com **degradação graciosa** (outbox/retry/fila Sidekiq; piso 10). Só depois o paralelo volta a `nproc`.

## 7. Integração com o resto da casa

Comando: **`/<slug>-ops`** (ex.: `/eng-ops`, `/ruby-ops`) — audita/scaffolda o ops, verifica o fluxo de ambientes, o layout `/<app>/` + seed, o isolamento por usuário (Puma/Sidekiq), a paralelização (`nproc`) e a independência.
