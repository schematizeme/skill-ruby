# Testes — recorte Ruby

> **PONTEIRO, não cópia.** A **disciplina de teste** da casa é da **`schematize-qa`**: a pirâmide,
> teste de COMPORTAMENTO (não "renderizou"), o "verde de verdade" (smoke com asserção de conteúdo +
> assertion negativa + self-check que força uma falha conhecida), cobertura útil, a11y, regressão
> visual, contrato/dados, **flaky** (quarentena com prazo e dono), o fluxo **plan-first**
> (`/qa-plan` → `/qa-run`) e os **gates de CI que travam o merge**. Leia
> `schematize-qa` → `references/estrategia.md`, `references/categorias.md`,
> `references/execucao.md` e `references/flaky.md`.
>
> **Segurança ofensiva** (rejeição rota a rota, injeção/coerção, IDOR/BOLA, cross-tenant) é a
> **`schematize-pentest`** — não é Q.A. e não mora aqui.
>
> Aqui fica **só o que muda em Ruby**: o runner, a sintaxe, e as armadilhas do dialeto.
>
> *(Este arquivo e a antiga reference *testes-execucao* eram, juntos, ~450 linhas por skill — 66% já
> duplicado na `schematize-qa`, 23% que pertence à `schematize-pentest` e ~2% idiomático de
> verdade. Deriva por cópia foi o achado da Classe C/D da vistoria de 2026-08-21.)*

## O runner e o comando

```bash
bundle exec rspec                      # suíte
bundle exec rspec --seed 1234          # ordem aleatória reproduzível
COVERAGE=true bundle exec rspec        # SimpleCov
bundle exec rspec --bisect             # acha o teste que polui o estado
```

## O que muda de forma em Ruby

- **Ordem aleatória é piso** (`config.order = :random`), e o **seed vai no output**. Suíte que só
  passa em ordem fixa tem acoplamento por estado global — e `--bisect` acha o culpado.
- **`let` é lazy, `let!` é ansioso.** Trocar um pelo outro muda o que existe no banco na hora do
  exemplo; bug de teste que parece bug de código quase sempre mora aí.
- **`build_stubbed` > `create`** nas factories: `create` bate no banco e é a razão nº 1 de suíte
  lenta. Reserve `create` para o que precisa de persistência de verdade.
- **`travel_to` / `freeze_time`** do ActiveSupport em vez de `sleep`; e sempre com bloco, para não
  vazar o congelamento para o próximo exemplo.
- **Transactional fixtures não cobrem thread separada** (Sidekiq inline, Capybara com servidor):
  aí use truncation, ou o teste enxerga um banco vazio e passa por engano.
- **`ActiveJob::TestHelper`/`Sidekiq::Testing.fake!`** para assertar **enfileiramento**; o
  `perform_now` no meio do teste esconde o contrato assíncrono.
- **Property-based:** `rantly`. **Mutation:** `mutant`. **AppSec estático:** `brakeman` e
  `bundler-audit` no mesmo gate.

## Onde divergir da base, a base manda

O piso é o mesmo: teste é **visto falhar no vermelho** antes de valer; cobertura é **contrato**
(não se baixa a régua para passar o CI); **teste nunca dispara efeito externo real** — endereço no
domínio de teste em rota nula, provider = sink, cap por execução, e a caixa se confere **lendo do
sink** (`references/iam.md` §3.1 desta skill; normativa em `schematize-engineering` →
`references/efeitos-externos.md`); e **gate não se desliga "por enquanto"**.
