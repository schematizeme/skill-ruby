# Cadeia de Suprimentos de Software (supply chain)

> Piso de **segurança da cadeia de suprimentos** — consolida num só lugar o que
> antes ficava espalhado (higiene de dependência em `seguranca.md`, imagem/deploy
> em `operacao.md`). O ataque hoje raramente é no seu código: é numa gem,
> numa imagem base ou no pipeline. Trate build e dependências como superfície de
> ataque de primeira classe.

## Índice
- S1. Dependências
- S2. SBOM e vulnerabilidades
- S3. Imagem de container
- S4. Pipeline e proveniência (SLSA)
- S5. Segredos no build

---

## S1. Dependências

**MUST**
- **`Gemfile.lock` commitado é piso** — build **reproduzível** a partir dele.
  `bundle install --frozen` (ou `--deployment`) no CI e em produção: **falha** se o
  lock divergir do `Gemfile`. Sem range frouxo (`>= x`) em gem sensível; pinar versão.
- **Verificar nome, licença e versão** de toda gem nova: typosquatting no rubygems.org
  é real (nome quase igual ao popular); licença na allowlist (MIT/Apache-2.0/BSD/MPL-2.0/
  ISC ok; GPL/AGPL/SSPL/proprietária só com ADR).
- **Minimizar superfície:** menos gem transitiva é menos risco. Avalie peso e
  manutenção antes de adicionar; corte o que não usa.
- **`.ruby-version` pinado** e dentro do suporte — Ruby EOL fora do build (liga com
  `stack-versoes.md`).

**SHOULD**
- Atualização automatizada (Dependabot/Renovate) com CI verde como gate.
- Mirror/proxy de gems (Gemfury/Artifactory) para builds críticos não dependerem do
  rubygems.org no ar; `bundle config` para gems privadas com source pinada.

**VETADO**
- Instalar de fonte não confiável / `gem install` fora do bundle ou `curl | sh` de
  origem não verificada no build.
- Fixar gem por branch/git ref móvel (`branch: main`) em produção.

---

## S2. SBOM e vulnerabilidades

**MUST**
- **Gerar SBOM** (CycloneDX via **`cyclonedx-ruby`**) no build e **versioná-lo junto ao
  artefato** — é o inventário do que foi de fato embarcado.
- **Scan de vulnerabilidade no CI, travando o merge** em `high`/`critical` sem ADR de
  aceite: **`bundler-audit`** (CVE de gem, base do rubysec) e **`brakeman`** (SAST Rails,
  trava em achado alto). Scan também na imagem (`grype`/`trivy`).
- Vulnerabilidade aceita conscientemente vira ADR com prazo de correção, não um
  silêncio.

**SHOULD**
- `bundler-audit update` no pipeline pra refrescar a base de advisories antes do scan.
- Verificação de licença de gem num gate só (advisories + licença).

---

## S3. Imagem de container

**MUST**
- **Base mínima e pinada por digest** (`ruby:<versão>-slim@sha256:...`), não por tag
  móvel (`latest`/`3`). Distroless/Alpine/scratch quando viável; menos pacote, menos CVE.
- **Não-root, filesystem read-only**, sem capabilities desnecessárias (liga com o
  piso de container do `operacao`/segurança).
- **Multi-stage build:** toolchain e `bundle install` só no estágio de build; a imagem
  final carrega só a app, as gems necessárias e o runtime Ruby — sem headers de compilação.

**SHOULD**
- Rebuild periódico pra absorver patch da base; recscan da imagem publicada.

---

## S4. Pipeline e proveniência (SLSA)

**MUST**
- **Build só no CI** (não na máquina do dev pra produção), a partir do código
  versionado; o pipeline é parte da TCB (base de confiança).
- **Assinar o artefato/imagem** (cosign/sigstore) e **verificar a assinatura na
  admissão** (deploy só aceita imagem assinada por pipeline confiável). Assinar sem
  verificar não protege.
- **Proveniência:** emitir atestação de build (quem buildou, de qual commit, com
  qual SBOM) — alvo **SLSA** crescente. Tag de imagem inclui o commit.

**SHOULD**
- Permissões mínimas no CI (token escopado, sem segredo amplo); branch protegida e
  review obrigatório antes de buildar release.

---

## S5. Segredos no build

**MUST**
- **Nenhum segredo no artefato/imagem** nem em layer intermediária (layer é
  inspecionável). Segredo de build via mecanismo efêmero (BuildKit secret mount),
  nunca `ARG`/`ENV` persistido, nunca `COPY .env` nem `config/master.key`/
  `RAILS_MASTER_KEY` embutida na imagem.
- **Secret scan no pipeline** (gitleaks/trufflehog) travando vazamento antes do
  merge. `.env` real e `config/master.key` fora do git.

> Regra de bolso: **você entrega tudo que embarca.** `Gemfile.lock` + SBOM + scan que
> trava (`bundler-audit` + `brakeman`) + imagem mínima/pinada/assinada + build no CI sem
> segredo no layer. Se não dá pra dizer exatamente o que tem dentro do artefato e quem o
> produziu, a cadeia está aberta.
