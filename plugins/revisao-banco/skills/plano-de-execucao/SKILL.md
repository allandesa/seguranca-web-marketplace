---
name: plano-de-execucao
description: >
  Mede o custo real de uma query no banco: lê EXPLAIN / EXPLAIN ANALYZE / plano de execução
  (PostgreSQL, MySQL/MariaDB, SQL Server, Oracle, SQLite), acha scan completo, nested loop por linha,
  ordenação em disco, estimativa errada e índice que falta, e usa pg_stat_statements, slow query log e
  Query Store para achar as queries que mais pesam. Complementa a skill revisao-queries, que revisa o
  código sem banco. Use quando houver acesso a um banco (cópia, homologação, réplica de leitura) e
  pedirem "por que esta query está lenta", "explica esse EXPLAIN", "qual índice criar", "quais queries
  pesam mais", "confirma se essa subquery é executada por linha", ou para fechar um item marcado
  "a medir" na revisão.
---

# Plano de execução

## Regras de segurança

- **Leitura só.** `EXPLAIN` sem `ANALYZE` não executa a query. `EXPLAIN ANALYZE` **executa**: nunca em
  `INSERT`/`UPDATE`/`DELETE`/`MERGE` fora de transação com `ROLLBACK`, e nunca em produção sem quem
  responde pelo banco autorizar.
- Prefira réplica de leitura ou cópia com volume parecido com o de produção. Plano medido em base de
  teste pequena engana: com 100 linhas, scan completo é o plano certo.
- Ponha tempo limite na sessão (`SET statement_timeout = '30s'` no PostgreSQL,
  `SET SESSION MAX_EXECUTION_TIME=30000` no MySQL) antes de medir query suspeita.
- Não crie índice no banco para testar sem autorização: proponha e meça na cópia.
- Dado real que aparecer na saída (CPF, nome, valor) não vai para o relatório.

## Como medir

| Banco | Plano estimado | Plano real |
|---|---|---|
| PostgreSQL | `EXPLAIN (FORMAT TEXT) <query>` | `EXPLAIN (ANALYZE, BUFFERS, VERBOSE) <query>` |
| MySQL 8 / MariaDB 10.1+ | `EXPLAIN FORMAT=TREE <query>` | `EXPLAIN ANALYZE <query>` (MySQL 8.0.18+); `ANALYZE <query>` (MariaDB) |
| SQL Server | `SET SHOWPLAN_XML ON` | `SET STATISTICS IO, TIME ON` + plano real (Include Actual Execution Plan) |
| Oracle | `EXPLAIN PLAN FOR <query>` + `DBMS_XPLAN.DISPLAY` | `/*+ GATHER_PLAN_STATISTICS */` + `DBMS_XPLAN.DISPLAY_CURSOR(format => 'ALLSTATS LAST')` |
| SQLite | `EXPLAIN QUERY PLAN <query>` | idem (não há tempo real por nó) |

Use os parâmetros reais de um caso ruim (o maior cliente, o período máximo), não os do teste.

## O que procurar no plano

| Sinal | PostgreSQL | MySQL | SQL Server | Leitura |
|---|---|---|---|---|
| Scan da tabela inteira | `Seq Scan` com muitas linhas | `type: ALL`, `Table scan` | `Table Scan`/`Clustered Index Scan` | filtro não usa índice, ou não há filtro |
| Subquery por linha | `SubPlan` executado `loops=N` | `DEPENDENT SUBQUERY` | `Nested Loops` com subárvore repetida | correlacionada não decorrelacionada |
| Loop aninhado caro | `Nested Loop` com `loops` alto no lado de dentro sem índice | `Block Nested Loop`/hash em tabela grande | `Nested Loops` com `Number of Executions` alto | falta índice na chave do JOIN |
| Estimativa errada | `rows=` estimado muito diferente do real | `rows` x `filtered` contra o real | Estimated vs Actual Rows | estatística velha (`ANALYZE`), correlação entre colunas, função no predicado |
| Ordenação em disco | `Sort Method: external merge` | `Using filesort` com tabela grande, `Using temporary` | `Sort` com spill (aviso) | falta índice na ordem, ou `work_mem` |
| Leitura de muita página para pouca linha | `Buffers: shared read` alto, `Rows Removed by Filter` alto | `rows examined` >> `rows sent` | `logical reads` alto | índice pouco seletivo ou ausente |
| Lookup por linha | `Index Scan` + heap fetch em muitas linhas | `Using index condition` sem cobertura | `Key Lookup` em loop | índice de cobertura (`INCLUDE`) |

Regra de bolso: compare **linhas lidas** com **linhas devolvidas**. Ler 2 milhões para devolver 50 é o
achado, seja qual for o nome do nó.

## Achar o que mais pesa (sem olhar código)

- PostgreSQL: `pg_stat_statements` ordenado por `total_exec_time` e por `calls`; `auto_explain` com
  `log_min_duration` para capturar planos lentos.
- MySQL/MariaDB: slow query log (`long_query_time`), `performance_schema.events_statements_summary_by_digest`,
  `sys.statements_with_full_table_scans`.
- SQL Server: Query Store (top consumers), `sys.dm_exec_query_stats`.
- Oracle: AWR/ASH, `V$SQL` por `elapsed_time`.

Muitas chamadas idênticas e rápidas (milhares por minuto com o mesmo formato) são sinal de N+1 na
aplicação, não de query lenta: volte para `revisao-queries` e procure o loop.

## Índice: propor e provar

1. Escreva o índice que atende o predicado: igualdade primeiro, depois faixa, depois ordenação; colunas do
   SELECT em `INCLUDE` se quiser cobertura.
2. Meça antes e depois na cópia: linhas lidas, buffers/leituras lógicas, tempo.
3. Diga o custo: tamanho do índice e o peso na escrita da tabela.
4. Em produção, crie sem travar (`CONCURRENTLY`/`ONLINE`), ver `revisao-queries/references/escrita-e-migrations.md`.

## Relatório

Para cada query: o texto (sem dado real), o parâmetro usado, o plano resumido (nó caro, linhas lidas x
devolvidas, tempo), o diagnóstico em uma frase e a correção com a medida de antes e depois. O que não foi
medido fica como **não medido**, nunca como "ok".
