# operacao — recorte Ruby

> **PONTEIRO, não cópia.** A normativa deste tema é da base: **`schematize-engineering`** →
> `references/operacao.md`. Leia lá primeiro; aqui fica **só o que muda em Ruby**.
>
> **Onde este arquivo divergir da base, a BASE MANDA** (`SKILL.md` §"Precedência e herança").
> Este arquivo era um clone com **88%** de conteúdo
> idêntico à base — deriva por cópia foi o achado da Classe C da vistoria de 2026-08-21, e ela
> já tinha atingido piso de segurança (o `argon2id-only` da casa virou "ou PBKDF2" numa skill,
> o rol de 6 linguagens virou "só Go e Rust" em três). Manter uma cópia é manter a próxima deriva.
## 20. Configuração

- Via environment variables (12-factor); nada de `Rails.env`-switch escondendo config.

- Validação tipada no boot — o processo carrega, valida o schema de config (ex.: `dry-schema`/`dry-validation` ou objeto de config próprio) e **falha rápido** se faltar variável ou o tipo estiver errado. Nunca descobrir config faltando no meio de um request.

- Sem hardcode; nada de segredo em `config/secrets.yml`/`credentials.yml.enc` versionado com chave commitada.

## 21. Infraestrutura e Deploy

- **Imagem de container:** base **Ruby slim pinada por digest** (não por tag móvel), usuário **não-root**, filesystem **read-only**. Dependências instaladas com `bundle install --deployment --frozen` (gems congeladas pelo `Gemfile.lock`, sem resolver no deploy). O artefato carrega **proveniência git** (commit SHA embutido em label/ENV) — é o mesmo bit que rodou no teste local.

- **Runtime:** a app Ruby sobe via **Puma** sob **systemd**; **Sidekiq** roda como **unit systemd à parte** (worker isolado, ver `references/ops.md` §3). Web e worker escalam e falham independentemente.

- **Migration é reversível por piso:** ActiveRecord `change` (reversível automático) ou par `up`/`down` explícito; toda migration é **testada com rollback** (`rails db:migrate` depois `rails db:rollback`) antes do merge. Migration sem `down` que funcione = defeito (§35).

### 21.1 Estratégia de Deploy

- Healthcheck gating: tráfego só vai pra pod ready (`/healthz`/`/readyz` servidos pelo Puma).

## 24. Qualidade e Git

- ≥ 1 reviewer (≥ 2 para `domain`, schema/migration, segurança).

- CI verde obrigatório (RuboCop + brakeman + bundler-audit + rspec — §34).

## 26. Runbooks e Incidentes

### 26.1 Runbooks

- Conteúdo mínimo: como diagnosticar falhas comuns, dashboards relevantes, comandos úteis (`bundle exec rails runner`, inspeção de fila Sidekiq, `bin/` scripts), como fazer rollback, contatos.

## 27. ADR — Architecture Decision Records

**Quando criar:** escolha de banco/broker/gem estrutural/framework (Rails vs Sinatra vs Hanami), padrão arquitetural, mudança de contrato público, qualquer desvio deste documento (exceto itens VETADO, que não admitem exceção).

## 28. Archive de Conversas e Tarefas — INEGOCIÁVEL

> **Esta seção não tem modo "pula pra ir mais rápido".** O archive é parte da entrega, não um extra. Tarefa sem archive = tarefa não feita (§35). Gerar os `.md` é tão obrigatório quanto o `bundle exec rspec` passar.

### 28.0 Layout canônico — todo MD gerado no archive, root limpo (MUST)

**Todo `.md` gerado pela skill/agente mora em `<projeto>_archive/`, NUNCA no root do projeto.** Isso vale para MAPA, índices, planos, relatórios, handoffs, checkpoints — qualquer artefato gerado. O root do projeto fica **limpo**: só código, config e os poucos MDs de projeto mantidos à mão por humano (`README.md`, `CLAUDE.md`, `LICENSE`, e o `CHANGELOG.md`). Largar MAPA/índice/plano/relatório no root é **violação** (§37) e fere a contenção de workspace.

Subpastas canônicas (o archive **é versionado** — entra no PR):
