# Subqueries

## Subquery sempre pesa? Não.

O que pesa é a subquery **que roda uma vez por linha** da query de fora. A mesma ideia escrita de outro
jeito pode custar uma leitura só.

| Forma | Custo típico | Revisão |
|---|---|---|
| Não correlacionada: `WHERE cliente_id IN (SELECT id FROM cliente WHERE uf = 'SP')` | roda uma vez; o otimizador costuma virar semi-join | ok |
| `EXISTS (SELECT 1 FROM item i WHERE i.pedido_id = p.id)` com índice em `item.pedido_id` | semi-join, para no primeiro achado | ok, e muitas vezes melhor que `JOIN` + `DISTINCT` |
| Escalar não correlacionada: `WHERE data > (SELECT MAX(data) FROM fechamento)` | roda uma vez | ok |
| **Escalar correlacionada no SELECT**: `SELECT p.id, (SELECT SUM(valor) FROM item i WHERE i.pedido_id = p.id) FROM pedido p` | um agregado por linha devolvida | **bloqueia** em listagem |
| **Várias correlacionadas no WHERE**: `WHERE (SELECT COUNT(*) ...) > 0 AND (SELECT MAX(...) ...) < x` | cada uma por linha, e o otimizador desiste de reordenar | **bloqueia** |
| `NOT IN (SELECT coluna ...)` em coluna que aceita NULL | se a subquery devolver um NULL, o resultado é vazio | **bloqueia** (resultado errado) |
| Subquery na condição do `JOIN ... ON` ou dentro de `CASE` por linha | por linha | bloqueia em listagem |

O otimizador às vezes salva a escalar correlacionada (o PostgreSQL e o SQL Server desaninham alguns
casos; o MySQL 8 melhorou muito em relação ao 5.7; o SQLite quase nunca). Não conte com isso: o mesmo
código que passa no PostgreSQL pode cair num MySQL 5.7, e uma versão nova do planner pode mudar. Se a
query é de listagem, peça a forma que não depende da sorte.

## Por que a escalar no SELECT é a pior

`SELECT id, (SELECT SUM(valor_total) FROM venda_produto vp WHERE vp.venda_id = v.id) FROM venda v`

- Para 50 linhas devolvidas, são 50 agregações. Para uma exportação de 200 mil vendas, 200 mil.
- Cada coluna calculada assim é mais uma rodada por linha: três colunas, três vezes.
- Ela esconde o custo: a query parece uma só no log e no código.
- Ela trava a paginação: o `LIMIT` às vezes só é aplicado depois de calcular tudo (com `ORDER BY` na
  coluna calculada, sempre).

## Reescritas

**1. Agregar uma vez e juntar** (a mais portável):

```sql
SELECT v.id, COALESCE(t.total, 0) AS total
FROM venda v
LEFT JOIN (
  SELECT venda_id, SUM(valor_total) AS total
  FROM venda_produto
  WHERE venda_id IN (/* mesmo filtro da query de fora, ou um JOIN com ele */)
  GROUP BY venda_id
) t ON t.venda_id = v.id
WHERE v.data BETWEEN :ini AND :fim
```

Repita o filtro de fora dentro da derivada (ou faça o `JOIN` antes do `GROUP BY`). Sem isso, a derivada
agrega a tabela inteira para usar 50 linhas.

**2. `JOIN` + `GROUP BY` direto**, quando há uma tabela filha só:

```sql
SELECT v.id, SUM(vp.valor_total) AS total
FROM venda v
LEFT JOIN venda_produto vp ON vp.venda_id = v.id
WHERE v.data BETWEEN :ini AND :fim
GROUP BY v.id
```

Cuidado com duas filhas no mesmo `JOIN` (itens e pagamentos): o produto cartesiano multiplica as somas.
Nesse caso, uma derivada agregada por filha (reescrita 1).

**3. `LATERAL` / `CROSS APPLY`**, quando é "o último registro de cada um" ou um top-N por linha, com
`LIMIT` dentro e índice na chave + ordem:

```sql
SELECT c.id, u.data, u.valor
FROM cliente c
LEFT JOIN LATERAL (
  SELECT data, valor FROM pedido p WHERE p.cliente_id = c.id ORDER BY data DESC LIMIT 1
) u ON true
WHERE c.id = ANY(:ids)
```

Ainda é por linha, mas com índice `(cliente_id, data DESC)` é uma busca de índice por linha, e só para as
linhas já filtradas. Aceitável com a lista de fora limitada.

**4. Função de janela**, para "o último de cada" ou ranking sem subquery:

```sql
SELECT * FROM (
  SELECT p.*, ROW_NUMBER() OVER (PARTITION BY cliente_id ORDER BY data DESC) AS rn
  FROM pedido p WHERE p.data >= :ini
) x WHERE rn = 1
```

**5. Várias subqueries no WHERE** viram `EXISTS` com índice, ou um `JOIN` com uma derivada agregada uma
vez só, com o filtro aplicado depois (`HAVING` ou `WHERE` na derivada).

**6. `NOT IN` com NULL** vira `NOT EXISTS`:

```sql
-- errado se pedido.cliente_id aceitar NULL: volta vazio
SELECT * FROM cliente WHERE id NOT IN (SELECT cliente_id FROM pedido)
-- certo
SELECT * FROM cliente c WHERE NOT EXISTS (SELECT 1 FROM pedido p WHERE p.cliente_id = c.id)
```

## No ORM

O mesmo padrão aparece disfarçado:

- Campo calculado/`@Formula`/`annotate(Subquery(...))`/`withCount`/`withSum` é subquery escalar por linha.
  Em listagem pequena e paginada é aceitável; em exportação, não.
- Propriedade computada no model que faz query (`get total() { return db.item.sum(...) }`) chamada no
  serializer é **N+1**, ver `n-mais-1.md`.

## Como escrever o achado

- Diga quantas vezes a subquery roda: "uma por linha devolvida; a lista não tem limite, então uma por
  venda do período".
- Proponha a reescrita pelo nome (agregar e juntar, LATERAL, janela, NOT EXISTS), não reescreva a query.
- Subquery não correlacionada só é achado com plano na mão (marque **a medir**).
