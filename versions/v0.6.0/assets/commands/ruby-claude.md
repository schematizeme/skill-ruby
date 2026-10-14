---
description: schematize-ruby — cria ou ATUALIZA (sobrescreve) o CLAUDE.md da raiz do repositório com a versão atual da skill; faz backup se houver customização local
---

Sincronize o **`CLAUDE.md` da raiz** deste repositório com a versão **atual** da skill `schematize-ruby`.

1. Localize o `assets/CLAUDE.md` da skill: `.claude/skills/schematize-ruby/assets/CLAUDE.md` (instalação no projeto) ou `~/.claude/skills/schematize-ruby/assets/CLAUDE.md` (global).
2. **Repo multi-linguagem:** se o `CLAUDE.md` da raiz já contém o bloco de **outra skill** (go/rust/web/node/engineering), **não sobrescreva cego** — os `CLAUDE.md` são complementares por fronteira (cada um governa seu lado). **Mescle por seção** (adicione/atualize só o bloco desta skill) e avise.
3. **Se existe só o desta skill:** compare com o da skill.
   - Se for uma versão antiga do template (sem customização), **sobrescreva** pela atual.
   - Se tiver customização local (seções que você/o usuário adicionaram fora do template), **preserve**: salve `./CLAUDE.md.bak` com o conteúdo antigo, aplique o template novo e reaplique as customizações por cima — avisando o que mudou.
4. **Se não existe:** crie `./CLAUDE.md` a partir do `assets/CLAUDE.md` da skill.
5. Confirme ao usuário: caminho, se **sobrescreveu** ou **mesclou**, e se gerou backup.

Este é o jeito **explícito de atualizar um `CLAUDE.md` que já existe** — rodar não pode deixar a versão antiga. Em repo com várias linguagens do rol, rode também o `/<slug>-claude` de cada lado (este é o de backend Ruby; o frontend vem do `schematize-web`; a base agnóstica é a `schematize-engineering`).
