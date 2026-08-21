# Padrões de Código — limites, granularidade, comentários e MAPA

Piso normativo de **organização do código** para Ruby, derivado da base agnóstica
(`schematize-engineering`) e válido para toda skill da casa. É inegociável: vale para
código humano e gerado por IA. O gate `/ruby-review` (DoD) reprova o que violar isto.
O enforcer de estilo/complexidade é o **RuboCop** (`.rubocop.yml`) — os cops `Metrics/*`
mapeiam os limites abaixo.

## 1. Tamanho de arquivo — teto de 750 linhas (≤ 500 de código útil + ~250 de comentário)

Regra **em camadas**: um teto duro generoso e um **flag** mais cedo que sinaliza
cheiro de classe/método extenso. A ideia é dar espaço pra código bem documentado sem
soltar a mão da granularidade.

- **Teto DURO: 750 linhas por arquivo `.rb`** (sem contar `# frozen_string_literal`,
  `require`s e licença no topo). Desse orçamento, **~250 linhas são reservadas a
  comentário/YARD** (§3) e **até ~500 são código útil**. Passou de **750 no total** — ou
  de **~500 de código útil** — → **quebre em mais de um arquivo**, por coesão (uma
  classe/módulo por arquivo), nunca por corte arbitrário no meio de um método. O gate
  `check-diff` (§6) **BLOQUEIA** acima do teto.
- **FLAG em > 300 linhas de código útil** (não bloqueia, mas **sempre sinaliza**):
  se o **código útil** de um arquivo — descontando comentário e linha em branco —
  passar de **300 linhas**, isso é **indício** de que a classe/método está **muito
  extenso** e provavelmente **pode ser melhorado ou precisa de um nível de abstração
  maior** (fat model, god-object, service inchado). Não trava a entrega; o gate
  **flagueia sempre** e o ponto entra como **dívida técnica pra rever quando as
  prioridades forem resolvidas** (registre no archive/ADR). **Nunca engula o flag.**
- **Observabilidade tem folga natural (~400 úteis):** arquivos dominados por
  instrumentação (tracing, métricas, logging estruturado) incham por natureza — ~400
  linhas úteis é esperado. Pra esses, o flag só é significativo acima de **~400** de
  código útil (ainda assim registrado). O teto duro de 750/500 continua valendo igual.
- **Escopo: código-fonte Ruby** (`.rb`, `.rake`). **Fora do escopo** (não disparam o
  gate): specs/factories, migrations, `db/schema.rb`, código gerado por generator,
  fixtures, documentação/Markdown, config e lockfiles (`Gemfile.lock`) — seguem bom
  senso de tamanho, mas o gate mede **código**, não texto.
- Quebrou um arquivo? Atualize o **MAPA** (§4) no mesmo PR.

## 2. Uma classe/módulo por arquivo (lógica)

- A regra é **uma classe/módulo público (“unidade lógica”) por arquivo**, espelhando
  o path (`app/application/create_order.rb` → `Application::CreateOrder`). O arquivo
  existe para entregar aquela unidade; auxiliares privados pequenos da mesma unidade
  podem conviver, desde que sirvam só a ela e o arquivo siga o teto (§1).
- **Métodos pequenos.** Ideal ≤ 15 linhas (`Metrics/MethodLength`), responsabilidade
  única. Método com **> 300 linhas de código útil** — ou uma classe que estoura o teto —
  dispara o flag (§1): quebre em métodos privados nomeados com propósito único, ou
  extraia um **service object / PORO**. Se não for a prioridade agora, **flagueia e
  revisita depois**; o teto duro continua 750/500.
- Sem **fat model** e sem **god-object**: regra de negócio não mora no model
  ActiveRecord nem no controller — vai pra **service object** (`Application::*`) e
  **POROs** de domínio. Sem `app/services/misc.rb` ou `lib/utils.rb` balaio que acumula
  métodos sem relação. Nome do arquivo = o que ele faz.

## 3. Tudo comentado: motivo + comportamento esperado (YARD)

Toda **classe/módulo público** e **método público** carrega um **doc-comment YARD**
(`# @param`, `# @return`, `# @raise`) que responde, no mínimo:

- **Por quê existe** — o problema que resolve / a decisão que encapsula.
- **Como se espera que funcione** — o passo-a-passo em uma frase, pré-condições e
  invariantes.
- **Entradas** — cada `@param name [Tipo]`: o que é, faixa/validação esperada.
- **Saídas** — `@return [Tipo]` e seu significado; `@raise` possíveis e quando ocorrem.
- **Efeitos colaterais** — I/O, rede, banco (`INSERT`/`UPDATE`), enfileirar job, estado
  global, mutação.
- **Fluxo do dado (começo → meio → fim)** — **de onde vem** (qual origem/serviço/fila/
  tabela; se vem de **outra aplicação**, nomeie qual e o contrato/evento), **o que é
  feito** (a transformação/decisão), e **pra onde vai** (destino: qual serviço/tópico/
  tabela/resposta). Todo método que **cruza fronteira de aplicação** (chama, recebe de,
  ou notifica outro serviço) diz isso explicitamente — ninguém, humano ou agente, deve
  precisar caçar no código pra descobrir origem e destino do dado.

Exemplo mínimo (adapte ao método):

```ruby
# frozen_string_literal: true

# O quê:  valida o payload de checkout e cria o pedido.
# Onde:   use-case Application::CreateOrder; chamado pelo POST /v1/checkout
#         (Interface::CheckoutController).
# Efeitos: persiste em `orders` via OrderRepository; publica
#          catalog.order.created pelo outbox.
#
# @param params [CheckoutParams] campos já filtrados por strong params.
# @return [Domain::Order] pedido criado.
# @raise [Domain::InvalidCheckout] quando o carrinho é inválido.
def call(params) = Checkout.new(params).executar
```

> **O `...` não serve de "corpo omitido".** ✔ Verificado em 2026-08-21 (Ruby 3.4, via container):
> `def call(params) = ...` é **SyntaxError** (*unexpected end-of-input; expected an expression after
> the operator*). O `...` em Ruby é **argument forwarding** — `def encaminha(...) = destino(...)`,
> que é sintaxe válida —, não reticências de exemplo. Num arquivo normativo isso importa mais do que
> parece: **exemplo que não compila é copiado assim mesmo**, e o leitor perde tempo achando que
> errou o resto.

Comentário que só repete a assinatura não conta. O doc-comment é o contrato: quem chama
deve entender o método **sem ler o corpo**. O índice de microfunções (`/ruby-index`) é
**gerado** desses doc-comments (varre `rg -n '^\s*def '` mais as declarações de
`class`/`module`) e falha o CI se achar método público sem contexto. **Toda
funcionalidade é mapeada** (§4): nenhuma fica fora do índice/MAPA — busca burra que torra
token e tempo é sintoma de mapa incompleto, não de falta de busca.

**`# frozen_string_literal: true`** é obrigatório no topo de todo arquivo `.rb` — o
RuboCop (`Style/FrozenStringLiteralComment`) reprova a ausência.

**Metaprogramação que esconde comportamento é VETADA sem doc + teste.** `method_missing`,
`define_method` dinâmico, `const_missing`, `send` sobre input — se o comportamento não
aparece num `def` legível e indexável, ele foge do índice de microfunções (§39) e de
quem lê o stack trace. Se for realmente necessário, **documente em YARD o conjunto de
métodos gerados** e **cubra com teste** cada caminho; caso contrário, prefira métodos
explícitos. Tipagem gradual (**RBS + Sorbet**) é recomendada para código não-trivial e
liga com `references/stack-versoes.md`.

## 4. MAPA da aplicação (arquivo-guia obrigatório)

Todo projeto que segue esta skill mantém um **`MAPA.md`** em
**`<projeto>_archive/index/MAPA.md`** (template em `assets/MAPA.md`) — **nunca no root do
projeto**. Todo MD **gerado** (MAPA, índices, planos, relatórios, handoffs) mora no
archive; o root fica limpo (só código, config e os poucos MDs mantidos à mão: README,
`CLAUDE.md`, LICENSE). Layout canônico do archive em `references/operacao.md` (§28). É
parte da entrega, atualizado **no mesmo PR** que mexe no código (o archive é versionado).
Ele lista, para **cada** método/função — público e privado, **sem exceção** (uma entrada
por unidade):

- **Onde está** — caminho do arquivo (e símbolo `Classe#metodo`).
- **Para que serve** — propósito em uma linha.
- **Dependências** — o que ele chama (métodos/classes/gems/serviços externos).
- **Auxiliares** — quem o apoia / quem depende dele (chamadores).
- **Entrada e saída** — de onde vêm os dados e para onde vão (args/retorno, rota, fila
  Sidekiq, arquivo, tabela).

O MAPA tem duas camadas, espelhando o índice de funcionalidades:

- **Global** (mantido à mão): repos, pastas, packs, models, como se comunicam, pontos de
  entrada (rotas/jobs Sidekiq/rake/CLIs) e de saída (banco/fila/API externa).
- **Microfunções** (gerado por `/ruby-index` a partir dos doc-comments §3).

O índice de microfunções é **exaustivo e conferível por contagem**: **uma entrada por
método** de **cada** serviço/repo (público e privado) — o número de entradas **tem que
bater** com o número de `def`s do código (`nº entradas == nº métodos`). Menos que isso é
**falha**, não "resumo": um MAPA de 90 linhas para 100+ métodos está errado. E o MAPA é um
**grafo**, não uma lista — traz o **grafo de serviços** (quem chama/notifica quem, por
contrato) e o **grafo de chamadas** por método (quem chama / é chamado), como **Mermaid +
adjacência**, pra percorrer do ponto de entrada à saída e medir o raio de impacto.
Contrato e gate: `references/entrega.md` §39.

Sem MAPA atualizado, o PR não passa na DoD. O objetivo: qualquer pessoa (ou IA) abre o
MAPA e sabe **onde tocar, o que aquilo afeta e por onde entra/sai** antes de ler o código.

## Checklist (entra na Definition of Done)

- [ ] Nenhum arquivo `.rb` > 750 linhas (nem > ~500 de código útil) — teto duro.
- [ ] Arquivo/classe/método com > 300 linhas de código útil (~400 em observabilidade) **flagueado** e registrado como dívida pra rever (não bloqueia, mas nunca silenciado).
- [ ] Uma classe/módulo por arquivo; métodos pequenos (`Metrics/MethodLength`).
- [ ] `# frozen_string_literal: true` no topo de todo arquivo `.rb`.
- [ ] Toda classe/método público com doc-comment YARD (motivo, comportamento, entradas, saídas, efeitos).
- [ ] Sem metaprogramação mágica sem doc + teste; fat model/god-object extraídos em service objects/POROs.
- [ ] RuboCop verde (`.rubocop.yml`): estilo + `Metrics/*` (MethodLength, AbcSize, CyclomaticComplexity, BlockNesting).
- [ ] `MAPA.md` atualizado no mesmo PR, em **`<projeto>_archive/index/`** (nunca no root) — camada global à mão + microfunções gerada.
- [ ] Índice de microfunções **exaustivo**: uma entrada por método, `nº entradas == nº métodos` do serviço (`/ruby-index` **reprova** se faltar e lista os ausentes); nenhum órfão.
- [ ] **Grafo** presente: serviços (quem chama quem) + chamadas por método (Mermaid + adjacência).
