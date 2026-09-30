---
description: schematize-ruby — lista todos os comandos disponíveis e o que cada um faz
---

Mostre ao usuário a lista de comandos do conjunto **schematize-ruby**, em formato
de tabela legível, exatamente com este conteúdo (ajuste se houver comandos novos
instalados em `.claude/commands/`):

| Comando | O que faz |
|---|---|
| `/ruby-help` | Lista todos os comandos do schematize-ruby (este). |
| `/ruby-load` | **Carrega à força TODO o corpo normativo** (DDD/arquitetura, clean code, segurança, dados, concorrência/GVL, testes, operação) no contexto e passa a aplicá-lo no projeto como regra inegociável. |
| `/ruby-claude` | Cria ou **atualiza (sobrescreve)** o `CLAUDE.md` da raiz com a versão atual da skill (backup se houver customização local; mescla se houver bloco de outra skill). |
| `/ruby-cc` | Context compact: gera `context.md` + `checklist.md` em `<projeto>_archive/context/` e roda `/compact`. |
| `/ruby-handoff` | Gera o handoff (`context.md` + `checklist.md`) **sem** compactar — ideal pra fim de sessão ou troca de tarefa. |
| `/ruby-qa` | Q.A. no contexto Ruby (aplica a schematize-qa: /qa-plan → /qa-run) — plan-first com `rspec`/RSpec, pede aprovação e roda faseado/assistido ou de uma vez. |
| `/ruby-review` | Roda o gate da Definition of Done e dos anti-padrões (§35, §37): arquivo >750 linhas bloqueia / >300 úteis flag, método/classe sem doc-comment YARD, índice desatualizado, macaquices de segurança (SQL interpolado, `rand` pra token, `YAML.load`). |
| `/ruby-iam` | Força/audita/scaffolda o IAM da casa (identidade≠email, ≥2 fatores, ReBAC multi-tenant deny-default, sessão longa/logout irreversível) como **app Ruby separada** em `auth.<domain>`, ou porta um auth legado (prioridade 0). |
| `/ruby-index` | (Re)gera o índice de microfunções (§39) a partir dos doc-comments YARD dos métodos. |
| `/ruby-ops` | Audita/scaffolda o `<projeto>_ops` (interface única): fluxo de ambientes, instalação paralela (`nproc`), independência dos serviços. |

Depois da tabela, diga em uma linha que o detalhe normativo está na skill
`schematize-ruby` (referências em `references/`), que a base agnóstica é a
`schematize-engineering`, e que o site é `skills.schematize.me/ruby`.
