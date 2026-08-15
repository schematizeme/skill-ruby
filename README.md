# schematize-ruby

> Padrões normativos de engenharia da casa — recorte **Ruby** (Rails/Sinatra/Roda, gems, RSpec).
> Arquitetura, segurança, IAM, testes/pentest, dados, observabilidade, deploy e archive.
> Ruby é linguagem **sancionada** do rol (prototipagem, scripts/automação, DX de produto com
> Rails, legado Ruby) — escolhida por **fit + ADR**.

Pacote de **skill normativa para [Claude Code](https://claude.com/claude-code)**.
Parte do catálogo **schematize skills** ([skills.schematize.me](https://skills.schematize.me)).

## Instalar

### Última versão (recomendado)

A partir de um clone do repositório:

```bash
git clone https://github.com/schematizeme/skill-ruby.git
cd skill-ruby && ./install.sh            # instala no projeto atual (diretório corrente)
# ./install.sh /caminho/do/projeto        # ou aponte para outro projeto
```

Ou baixe o `.zip` da última release e descompacte direto em `.claude/skills/`:

```bash
curl -L -o schematize-ruby.zip \
  https://github.com/schematizeme/skill-ruby/releases/latest/download/skill-ruby.zip
unzip schematize-ruby.zip -d .claude/skills/
```

### Uma versão específica

Cada versão tem três formas de obter: **(1)** um Release com `.zip` para baixar,
**(2)** uma pasta navegável em `versions/`, e **(3)** uma tag git.

| Versão | Data | Download (.zip) | Pasta navegável | Notas |
|---|---|---|---|---|
| **0.1.0** | 2026-08-15 | [release](https://github.com/schematizeme/skill-ruby/releases/download/v0.1.0/skill-ruby.zip) | [versions/v0.1.0/](versions/v0.1.0) | [CHANGELOG](CHANGELOG.md) |

```bash
# clonar uma versão exata pela tag:
git clone --branch v0.1.0 https://github.com/schematizeme/skill-ruby.git
```

> Todas as versões aparecem na página de **[Releases](https://github.com/schematizeme/skill-ruby/releases)**.

## Comandos

Todos prefixados por `ruby-` — **sem conflito** com as outras skills na mesma máquina.

| Comando | O que faz |
|---|---|
| `/ruby-help` | lista todos os comandos do schematize-ruby |
| `/ruby-load` | carrega à força todo o corpo normativo no contexto |
| `/ruby-claude` | cria/atualiza o `CLAUDE.md` da raiz com a versão atual da skill |
| `/ruby-cc` | context compact: gera handoff no archive e roda `/compact` |
| `/ruby-handoff` | gera o handoff (context.md + checklist.md) **sem** compactar |
| `/ruby-qa` | fluxo de Q.A. plan-first (planeja, aprova, roda) |
| `/ruby-review` | roda o gate da DoD e dos anti-padrões no diff |
| `/ruby-iam` | força/audita/scaffolda o IAM da casa (auth como app separada) |
| `/ruby-index` | (re)gera o índice de microfunções a partir dos doc-comments YARD |
| `/ruby-ops` | audita/scaffolda o `<projeto>_ops` (interface única) |

Digite `/ruby-help` dentro do Claude Code para ver a lista completa.

## Conteúdo da skill

- `SKILL.md` — porta de entrada e pisos inegociáveis.
- `references/` — corpo normativo fatiado por domínio (leia o que casa com a tarefa).
- `assets/` — templates (ADR/TASK/RUNBOOK/…), comandos, `CLAUDE.md`, CI, lint, hooks.
- `scripts/` — andaime de testes, índice e gestão de contexto.
- `skill.toml` — manifesto da skill (slug, nome, versão, descrições).

## Skills irmãs

- [skill-go](https://github.com/schematizeme/skill-go) — backend Go (default pragmático).
- [skill-rust](https://github.com/schematizeme/skill-rust) — backend Rust (quando errar é caro).
- [skill-node](https://github.com/schematizeme/skill-node) — manutenção de legado Node/TS.
- [skill-web](https://github.com/schematizeme/skill-web) — frontend / SEO / performance.

Todas podem ficar habilitadas ao mesmo tempo: os comandos são namespaced por skill
(`ruby-*`, `go-*`, `rust-*`, `node-*`, `web-*`). A base agnóstica é a `schematize-engineering`.

## Licença

[MIT](LICENSE) © 2026 schematizeme.
