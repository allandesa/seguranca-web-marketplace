# Índices e predicados

Um filtro só ajuda se o banco conseguir usar índice nele. O predicado que não usa índice transforma a
consulta "com filtro" num scan da tabela inteira.

## Predicado que não usa índice (não sargável)

| Padrão | Exemplo | Troque por |
|---|---|---|
| Função na coluna | `WHERE DATE(criado_em) = :dia`, `YEAR(data) = 2025`, `UPPER(nome) = :n` | faixa: `criado_em >= :dia AND criado_em < :dia + 1`; índice funcional/coluna gerada para `UPPER`/`LOWER` |
| Conta na coluna | `WHERE valor * 1.1 > :x`, `WHERE data + INTERVAL '1 day' > now()` | mova a conta para o lado do parâmetro |
| Curinga no início | `LIKE '%abc%'`, `LIKE '%abc'` | prefixo `LIKE 'abc%'`; full-text ou trigram (`pg_trgm`) para "contém" |
| Conversão implícita | coluna `varchar` comparada com número; `cpf = 12345678900`; collation diferente no JOIN | compare com o mesmo tipo |
| `OR` entre colunas diferentes | `WHERE cliente_id = :c OR vendedor_id = :v` | `UNION ALL` de duas queries indexadas, ou índices separados que o banco combine |
| Negação | `WHERE status <> 'x'`, `NOT LIKE` | filtre pelo conjunto positivo, se for pequeno |
| `IS NULL` em coluna rara | depende do banco | índice parcial (`WHERE coluna IS NULL`) |
| Coluna fora de ordem no índice composto | índice `(empresa_id, data)` e filtro só por `data` | índice começa pela coluna de igualdade; depois a de faixa; depois a da ordenação |

## Índice que provavelmente falta

Sem acesso ao banco, aponte como **a medir**:

- coluna de FK usada em `JOIN` ou em `WHERE` (muitos bancos não indexam FK sozinhos, como o PostgreSQL);
- coluna de filtro obrigatório da tela (período, empresa/filial, status);
- combinação filtro + ordenação de lista paginada (`(empresa_id, data DESC, id DESC)`);
- coluna de busca do autocomplete.

E o contrário: índice novo em coluna de baixa seletividade (booleano, status com 3 valores) quase nunca
ajuda e pesa na escrita.

## Outros sinais

- `SELECT *` em listagem e em API: traz colunas grandes (texto, JSON, blob) e impede índice de cobertura.
  Peça as colunas usadas. Não bloqueia se tem limite.
- `DISTINCT` ou `GROUP BY` sem agregado para "tirar duplicata": o `JOIN` está errado ou deveria ser
  `EXISTS`.
- `COUNT(*) > 0` para existência: `EXISTS` ou `LIMIT 1`.
- `ORDER BY RAND()`/`NEWID()`/`random()`: ordena a tabela inteira.
- `ORDER BY` em coluna sem índice com `LIMIT`: o banco ordena tudo para devolver 20.
- `UNION` onde `UNION ALL` basta: ordena para tirar duplicata que não existe.
- `IN` com milhares de valores montados no código: lote, tabela temporária ou `= ANY(:array)`.
- Query montada por concatenação de string com valor do usuário: isso é injeção, não desempenho. Bloqueia
  e é assunto de segurança.
- JSON/JSONB filtrado sem índice (`dados->>'campo' = :x`): índice de expressão ou GIN.
