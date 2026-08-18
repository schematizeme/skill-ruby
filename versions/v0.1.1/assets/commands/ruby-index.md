---
description: (Re)gera o índice de microfunções (§39) a partir dos doc-comments YARD
argument-hint: "[dir de origem, ex: app ou lib]"
---

Atualize o **índice de funcionalidades** (§39) — **fonte da verdade** do projeto e base do MAPA (§4). O índice é **exaustivo**: **uma entrada por método/função**, de **cada** serviço, **com grafo**. Ele **enumera** o sistema; não resume.

## 1. Enumere e CONTE (a verdade do que existe)

Antes de gerar, descubra **todos** os métodos/funções do alvo `${ARGUMENTS:-app}` — públicos e privados, incluindo métodos de instância/classe, jobs (Sidekiq/ActiveJob), handlers/controllers e blocos nomeados. Conte as declarações (use AST/ctags se houver; senão, ripgrep):

- **Ruby:** `rg -n '^\s*def '` (métodos de instância e de classe `def self.`), mais `rg -n '^\s*(class|module) '` para as unidades.
- Em Rails, inclua actions de controller, jobs, mailers, service objects e models.

Guarde **N = total de métodos** por serviço/pasta. É o alvo de completude — e faça para **cada** serviço do sistema, não só o que você tocou.

## 2. Uma entrada por método (sem "relevante")

Para **cada** método encontrado, uma linha em `<projeto>_archive/index/INDEX_FUNCTIONS.md` (um arquivo por serviço):
`método | o quê | de onde vem → pra onde vai | chama (out) | é chamado por (in) | efeitos | arquivo:linha`.
Fonte: `scripts/build-index.mjs` se existir; senão, extraia dos doc-comments **YARD** (§3, `# @param`/`# @return` + O quê/Onde). **Nenhum** método fica de fora por "não ser relevante".

## 3. Construa o GRAFO (o mapa não é lista)

- **Grafo de serviços** → `INDEX_GLOBAL.md`: bloco `mermaid flowchart LR` com **todos** os serviços/apps como nós e arestas `A -->|contrato| B` (rota HTTP/evento/fila Sidekiq). Nenhum serviço de fora. Espelhe em adjacência textual (`A -> B (contrato)`) pra grep.
- **Grafo de chamadas** → por serviço, no `INDEX_FUNCTIONS.md` e no `MAPA.md` (§5): `mermaid flowchart` dos pontos de entrada (controller/job) às saídas **+** adjacência `chamador -> chamada`. Cada método é um nó.

## 4. Concilie a COMPLETUDE (gate duro)

- Conte as entradas do índice (**M**) e compare com **N**, **por serviço**.
- **Se M < N → FALHE.** Liste, **pelo nome**, os `N - M` métodos que ficaram de fora e volte ao passo 2 até `M == N`. Índice com menos entradas que métodos é bug, não resumo.
- Se `scripts/build-index.mjs` sair com código 1, há método sem doc-comment (§3): corrija **na origem** (o quê + de onde→pra onde), não no índice.

## 5. Global + MAPA + confirmação

- `<projeto>_archive/index/INDEX_GLOBAL.md`: **cada** repo/serviço com 1 linha, árvore de pastas top-level e o **grafo de serviços**. Nenhum de fora.
- Espelhe no `<projeto>_archive/index/MAPA.md` (§4) — **no archive, nunca no root**: microfunções geradas + grafos (§2.5, §5).
- Confirme ao usuário com **números**: `N métodos / M entradas / G serviços no grafo`, e que **M == N** em cada serviço. Se não bater, **não terminou**.
