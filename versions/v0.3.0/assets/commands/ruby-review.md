---
description: Roda o gate da Definition of Done e anti-padrões (§35, §37) no diff atual
argument-hint: "[ref git, ex: origin/main]"
---

Faça o **review de padrões** do diff atual (contra $ARGUMENTS ou `origin/main`),
combinando o checker determinístico com seu julgamento.

1. Rode `bash scripts/check-diff.sh ${ARGUMENTS:-origin/main}` e leia o resultado.
2. Some a isso a análise que o script NÃO faz bem sozinho:
   - **§37 (anti-padrões):** segredo no cliente/no Gemfile/no código (usar Rails credentials/ENV),
     **SQL interpolado em `where`/`find_by_sql`** (usar placeholders/hash), authz no client,
     `tenant_id` do body/params, JWT sem validar (ou `alg=none`), **`rand` pra token** (usar
     `SecureRandom`), **`YAML.load`/`Marshal.load`** em dado não-confiável (usar `YAML.safe_load`),
     **mass-assignment** sem strong params, `rescue nil`/`rescue => e; end` que engole erro,
     teste silenciado (`skip`/`pending`/baixar cobertura), metaprogramação mágica sem doc.
   - **§6:** arquivo >750 linhas (ou >~500 úteis) → bloqueia; >300 úteis → flag; método/classe sem
     doc-comment YARD (o quê + onde); mais de uma classe/módulo por arquivo; método enorme
     (`Metrics/MethodLength`).
   - **§39:** o índice de funcionalidades foi atualizado no mesmo PR?
   - **§3:** backend novo **fora do rol sancionado** ou sem ADR de fit? (Ruby é do rol, mas exige
     ADR justificando o encaixe; Node-backend/PHP novos são proibidos.)
   - **Cadeia:** `Gemfile.lock` commitado? `bundler-audit`/`brakeman` limpos? rodando em Ruby não-EOL?
3. Produza um relatório com `BLOQUEIA` (viola piso/DoD) e `ATENÇÃO` (melhorar),
   citando arquivo:linha. Se houver qualquer `BLOQUEIA`, a task **não está pronta** (§35).

Seja específico e acionável — aponte o conserto, não só o problema.
