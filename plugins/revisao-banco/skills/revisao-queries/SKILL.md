---
name: revisao-queries
description: >
  Revisa código que lê ou grava em banco de dados (SQL cru, query builder ou ORM) atrás do que derruba
  sistema em produção: consulta sem filtro nem limite, subquery correlacionada por linha, N+1 em loop,
  filtro de tela que carrega a tabela inteira (select de clientes, produtos, notas), exportação e
  relatório sem período, predicado que não usa índice, paginação por OFFSET profundo, UPDATE/DELETE
  sem WHERE, transação longa e migration que trava tabela grande. Agnóstica de banco (PostgreSQL,
  MySQL/MariaDB, SQL Server, Oracle, SQLite) e de stack (Prisma, TypeORM, Sequelize, Drizzle, Kysely,
  Knex, Eloquent, Doctrine, Django, SQLAlchemy, ActiveRecord, Hibernate/JPA, Entity Framework, Dapper).
  Use SEMPRE ao revisar PR, diff, endpoint, repositório, relatório, exportação, filtro ou migration que
  toque banco, quando pedirem "revisa essa query", "isso escala?", "por que está lento", "vai aguentar
  produção", e proativamente ao escrever listagem, busca, combo/select de cadastro, relatório ou job.
---

# Revisão de queries

## O que esta skill faz

Lê o código e aponta, com `arquivo:linha` e o trecho que prova, as queries que vão quebrar quando a
tabela crescer. O foco é o que derruba produção, não estilo. Funciona sem acesso ao banco: o que
depende de volume real ou de plano de execução sai como **a medir**, com o comando para medir
(ver a skill `plano-de-execucao`).

A pergunta de fundo para cada query é uma só: **quantas linhas ela lê e devolve quando a tabela tiver
10 milhões de linhas, e quantas vezes ela roda por requisição?** Uma query que lê tudo uma vez, ou que
lê pouco mil vezes, quebra do mesmo jeito.

## Princípios

1. **Evidência ou nada.** Todo achado cita `arquivo:linha` e copia o trecho. Sem trecho, não é achado.
2. **Volume decide a severidade.** Sem filtro numa tabela de configuração com 20 linhas não é problema.
   Sem filtro em venda, nota, log, movimento, evento ou cliente é. Na dúvida sobre o tamanho, trate como
   tabela que cresce e diga isso no achado.
3. **Subquery não é pecado; subquery por linha é.** Ver `references/subqueries.md`. Não peça para trocar
   `EXISTS` ou `IN (SELECT ...)` não correlacionado por `JOIN` sem motivo: o otimizador costuma fazer isso
   sozinho, e o `JOIN` pode até duplicar linha.
4. **Nada de lista sem teto.** Toda leitura que alimenta tela, API, exportação ou job tem filtro
   obrigatório, limite ou paginação, e o servidor impõe isso, não só a tela.
5. **O ORM não salva ninguém.** `findMany()`, `.all()`, `Model.objects.all()`, `->get()` sem `where` nem
   `take/limit` são `SELECT *` sem filtro. Lazy loading dentro de loop é N+1.
6. **Corrigir não é revisar.** Aponte o defeito e a correção em uma frase; reescreva a query só se pedirem.

## Fluxo

1. **Inventário.** Liste todo ponto do diff que toca banco: SQL em string ou arquivo `.sql`, chamadas de
   ORM e query builder, procedures, views, migrations, e também os pontos que **chamam** essas funções
   (o loop que chama o repositório é onde mora o N+1). Inclua endpoints de listagem, busca, autocomplete,
   relatório, exportação, dashboard, cron e fila.
2. **Para cada query, responda:**
   - Qual o filtro? Ele vem sempre, ou é opcional e pode chegar vazio?
   - Qual o teto de linhas devolvidas (LIMIT/TOP/FETCH/take/paginação)? Quem impõe: servidor ou tela?
   - Roda quantas vezes por requisição? Está dentro de loop, `map`, `forEach`, `Promise.all`, resolver de
     GraphQL, serializer ou template?
   - Tem subquery na lista do SELECT ou no WHERE que referencia a linha de fora (correlacionada)?
   - O predicado usa índice (coluna nua do lado esquerdo, sem função, sem cast, sem `LIKE '%...'`)?
   - Ordena e pagina de forma estável e barata?
   - Se grava: tem WHERE? Quantas linhas toca? Em lote ou de uma vez? Dentro de transação longa?
3. **Classifique** pela tabela de severidade abaixo.
4. **Relate** no formato do fim desta página.

Checklist completo por tema:

| Tema | Arquivo |
|---|---|
| Subquery escalar, correlacionada, `IN`, `NOT IN`, `EXISTS`, CTE, LATERAL | `references/subqueries.md` |
| Leitura sem filtro nem limite, filtro opcional, período obrigatório, exportação, combo/select de cadastro, autocomplete, contagem | `references/volume-e-filtros.md` |
| N+1 e query em loop, por ORM | `references/n-mais-1.md` |
| Predicado que não usa índice, `SELECT *`, `DISTINCT` remendo, `OR`, ordenação, paginação | `references/indices-e-predicados.md` |
| UPDATE/DELETE em massa, transação, lock, migration em tabela grande | `references/escrita-e-migrations.md` |

Para revisor automático (bot de PR, CI), use `checklist.md`: é a versão curta, só com o que bloqueia.

## Severidade

**Bloqueia** (quebra produção quando a tabela cresce, ou já quebra hoje):

- Leitura de tabela que cresce sem filtro obrigatório **e** sem limite: lista, exportação, relatório,
  dashboard, job. Inclui filtro que pode chegar vazio e virar "todos".
- Combo/select/dropdown que carrega o cadastro inteiro (clientes, produtos, fornecedores, notas) em vez
  de busca no servidor com mínimo de caracteres e limite.
- Relatório ou consulta sem período obrigatório, ou com período sem teto (ex.: "desde 2000").
- Subquery escalar correlacionada na lista do SELECT de uma listagem (um SELECT por linha devolvida).
- Várias subqueries correlacionadas no WHERE, ou `NOT IN (SELECT ...)` em coluna que aceita NULL (bug de
  resultado, não só de desempenho).
- N+1: query dentro de loop sobre resultado de outra query, lazy loading em loop, `Promise.all` com uma
  query por item sem teto.
- UPDATE ou DELETE sem WHERE, ou com WHERE que pode chegar vazio.
- UPDATE/DELETE/INSERT em massa numa transação só, sem lote, em tabela que cresce.
- Migration que trava tabela grande (índice sem `CONCURRENTLY`/`ONLINE`, `ALTER` que reescreve a tabela,
  `NOT NULL` ou `DEFAULT` volátil em coluna nova de tabela grande, backfill no mesmo passo).
- Paginação só na tela: a API devolve tudo e o front fatia.

**Não bloqueia** (efeito real, mas pequeno ou condicionado):

- `SELECT *` em query com limite; coluna a mais trafegando.
- Predicado que não usa índice numa tabela pequena ou já filtrada por índice seletivo.
- `OFFSET` profundo numa lista que raramente passa da página 10.
- `COUNT(*)` para saber se existe (troque por `EXISTS`/`LIMIT 1`).
- `DISTINCT` ou `GROUP BY` usado para esconder duplicata de JOIN.
- Índice que provavelmente falta: diga qual e marque **a medir**.

**A medir**: tudo que depende do plano real (o otimizador decorrelacionou? usou o índice?). Diga o
comando: `EXPLAIN (ANALYZE, BUFFERS)` no PostgreSQL, `EXPLAIN ANALYZE` no MySQL 8, plano real no SQL
Server. Detalhes na skill `plano-de-execucao`.

## O que não é achado

- Subquery não correlacionada (`WHERE id IN (SELECT ...)` sem referência à linha de fora), `EXISTS`
  com índice na chave, CTE simples: quase sempre bem otimizados. Só aponte com plano na mão.
- Tabela de domínio pequena e estável (UF, status, tipo) lida inteira, com cache ou não.
- Estilo de SQL, caixa alta, alias, ordem de cláusula.
- "Use stored procedure", "troque de ORM", "adicione cache" sem defeito concreto.

## Formato do achado

```
[BLOQUEIA] arquivo/caminho.ts:42: Lista de notas sem período nem limite
Trecho: const notas = await db.nota.findMany({ where: { empresaId } })
Impacto: devolve todas as notas da empresa desde o início; com 5 anos de emissão são milhões de linhas
         por chamada, a API estoura memória e o banco fica preso no scan.
Correção: exija período (máximo 31 dias) e pagine no servidor (take 100 + cursor).
```

Termine com a **cobertura**: quantos pontos de banco havia no diff, quantos foram lidos, e o que ficou
**a medir**. Não diga "as queries estão boas": diga o que foi conferido.
