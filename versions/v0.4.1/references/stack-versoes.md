# Anexo A — números voláteis (versões e limiares)

> Parte da skill **schematize-ruby**. **Fonte volátil** — versões e limiares mudam. Atualize aqui (revisão trimestral) sem mexer no corpo normativo. **Verificado em: 2026-08-21**. Sempre confirme o número atual antes de aplicar como gate.
> Conferido nas releases oficiais nesta data: a série corrente do Ruby é a **4.0** (último patch **4.0.6**, 14/07/2026). O piso normativo continua sendo *"série suportada, não-EOL"* — o número acima é calibração, não trava.

## Ruby runtime

- **Série suportada alvo:** rodar sempre em versão **suportada (não-EOL)**. Ruby faz **release anual** (~Natal); cada série major.minor recebe ~**3 a 3.5 anos** de manutenção (normal → security-only → EOL). Ruby **fora de suporte = piso de segurança violado (CVE sem patch)** (liga `seguranca.md`). Números exatos de série e datas mudam — confira em <https://www.ruby-lang.org/en/downloads/branches/>.
- `.ruby-version` **pinado** e dentro do suporte; imagem base **por digest** (`ruby:<versão>-slim`), nunca `latest`.
- Recursos que exigem versão mínima (piso antes de usar): `Fiber.scheduler` e **Ractor** (Ruby ≥ 3.0), `Data.define` (≥ 3.2), **YJIT estável** (≥ 3.3 — ligar em produção na série suportada).

## Rails

- Rodar em **série suportada**. ✔ **Corrigido em 2026-08-21:** a política do Rails é **por SÉRIE (`major.minor`), não por major** — correção de bug vai só para a **série estável mais recente**; correção de segurança vai para as **últimas séries** conforme a política vigente. Enunciar "as duas últimas majors" faz o time achar que uma `8.0` está coberta porque a `8.1` é a mesma major — e não está. Série fora da janela = EOL → **migrar** é a ação, não adiar. Corrente na verificação: **Rails 8.1** (8.1.3.1, 29/07/2026).
- Upgrade incremental pelo guia oficial; **dual-boot** com `next_rails` (Gemfile duplo) para migração gradual. `bundle exec rails app:update` reconcilia config.

## Tipagem

- **RBS** (assinaturas) + **Sorbet** (`sorbet`/`sorbet-runtime`, checagem gradual `# typed:`) — versões correntes, confirme. Assinatura de API pública e de fronteira de domínio ganha tipo; adoção incremental por arquivo.

## Limiares de dependências (gems — sinal de saúde, não veredito cego)

**Direto ≠ transitivo.**

| Sinal | Como medir | Limiar de *smell* |
|---|---|---|
| Gems diretas de produção | grupo default do `Gemfile` | **ordem de dezenas** → olhar, justificar cada família |
| Transitivas de produção | `bundle list` / `Gemfile.lock` | **ordem de centenas alta → reduzir/ADR** |
| Gems de dev/test | grupos `:development`/`:test` | sinal **separado** de supply-chain |

> Trate os números como **ordem de grandeza**, não gate exato. Calibre com serviços reais da casa e registre o limiar efetivo aqui.

## Licenças

- **Allowlist** (permitidas sem revisão): MIT, ISC, BSD-2/3-Clause, Apache-2.0, MPL-2.0, 0BSD.
- **Revisão obrigatória / evitar:** copyleft forte (GPL/LGPL/AGPL) em código distribuído; **gem sem licença** é bloqueada. Rodar `license_finder` no CI (liga `cadeia-suprimentos.md`).

## Ferramental (versões correntes — confirme)

`bundler`, `rubocop` (+ `rubocop-rails`/`-performance`/`-rspec`), `brakeman`, `bundler-audit`, `rspec`, `factory_bot`, `simplecov`, `sorbet`/`rbs`, `sidekiq`, `puma`, `lograge`/`semantic_logger`, `opentelemetry-sdk`/`opentelemetry-instrumentation-all`, `bullet`, `strong_migrations`, `cyclonedx-ruby`, `license_finder`.
