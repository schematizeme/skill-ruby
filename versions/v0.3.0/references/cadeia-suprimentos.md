# Cadeia de Suprimentos de Software (supply chain)


> **PONTEIRO, não cópia.** A normativa deste tema é da base: **`schematize-engineering`** →
> `references/cadeia-suprimentos.md`. Leia lá primeiro; aqui fica **só o que muda em Ruby**.
>
> **Onde este arquivo divergir da base, a BASE MANDA** (`SKILL.md` §"Precedência e herança").
> Em 2026-08-21 os blocos idênticos à base foram **podados mecanicamente** (`tools/podar-clone.mjs`),
> e é por isso que a numeração **salta**: o número é o da base, e o item ausente aqui é o que **não
> muda nesta linguagem** — procure-o lá.

## S1. Dependências

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

- Mirror/proxy de gems (Gemfury/Artifactory) para builds críticos não dependerem do
  rubygems.org no ar; `bundle config` para gems privadas com source pinada.

- Instalar de fonte não confiável / `gem install` fora do bundle ou `curl | sh` de
  origem não verificada no build.

- Fixar gem por branch/git ref móvel (`branch: main`) em produção.

## S2. SBOM e vulnerabilidades

- **Gerar SBOM** (CycloneDX via **`cyclonedx-ruby`**) no build e **versioná-lo junto ao
  artefato** — é o inventário do que foi de fato embarcado.

- **Scan de vulnerabilidade no CI, travando o merge** em `high`/`critical` sem ADR de
  aceite: **`bundler-audit`** (CVE de gem, base do rubysec) e **`brakeman`** (SAST Rails,
  trava em achado alto). Scan também na imagem (`grype`/`trivy`).

- `bundler-audit update` no pipeline pra refrescar a base de advisories antes do scan.

- Verificação de licença de gem num gate só (advisories + licença).

## S3. Imagem de container

- **Base mínima e pinada por digest** (`ruby:<versão>-slim@sha256:...`), não por tag
  móvel (`latest`/`3`). Distroless/Alpine/scratch quando viável; menos pacote, menos CVE.

- **Multi-stage build:** toolchain e `bundle install` só no estágio de build; a imagem
  final carrega só a app, as gems necessárias e o runtime Ruby — sem headers de compilação.

## S5. Segredos no build

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
