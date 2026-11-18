# Checklist de banco para revisão de PR

Versão curta da skill `revisao-queries`, para revisor automático. Vale para SQL cru, query builder e ORM,
e para o código que chama essas funções. Cada item é defeito só com a linha que prova; tabela pequena e
fixa (UF, status, tipo) lida inteira não é achado.

## Bloqueia

1. **Leitura sem teto.** Lista, busca, API, exportação, relatório, dashboard ou job que lê tabela que
   cresce (venda, nota, pedido, movimento, log, evento, cliente, produto) sem filtro obrigatório **e** sem
   limite imposto no servidor. Inclui filtro opcional que, vazio, vira "todos", e `findMany()`/`.all()`/
   `->get()` sem `where` nem `take`/`limit`.
2. **Período sem teto.** Consulta de dado temporal sem período obrigatório, ou com padrão amplo
   (`1900-01-01`, "desde sempre", fim opcional).
3. **Combo/select de cadastro inteiro.** Opções de filtro ou de formulário que carregam todos os clientes,
   produtos, fornecedores ou documentos, em vez de busca no servidor com mínimo de caracteres e limite.
4. **Paginação só na tela.** A API devolve o array completo e o front fatia; ou `pageSize` sem máximo no
   servidor.
5. **Subquery escalar correlacionada na lista do SELECT** de listagem ou exportação
   (`SELECT a.id, (SELECT SUM(...) FROM b WHERE b.a_id = a.id) FROM a`): um agregado por linha. Troque por
   agregação única com `JOIN`, `LATERAL` ou janela.
6. **Várias subqueries correlacionadas no WHERE**, ou **`NOT IN (SELECT ...)`** em coluna que aceita NULL
   (resultado vazio errado; use `NOT EXISTS`).
7. **N+1.** Query dentro de loop (`for`, `map`, `forEach`, `Promise.all`) sobre o resultado de outra query,
   lazy loading de relação em loop, serializer/resolver que consulta por item, gravação item a item.
8. **UPDATE/DELETE sem WHERE** ou com WHERE que pode chegar vazio (`updateMany({})`, `deleteMany({})`).
9. **Escrita em massa numa transação só**, sem lote, em tabela que cresce; transação aberta durante chamada
   externa.
10. **Migration que trava tabela grande:** índice sem `CONCURRENTLY`/`ONLINE`, troca de tipo, `NOT NULL`
    com default volátil, `FOREIGN KEY` validada na hora, backfill no mesmo passo.

## Não bloqueia (um achado por ponto)

- `SELECT *` em query com limite.
- Predicado que não usa índice (função na coluna, `LIKE '%x'`, cast implícito, `OR` entre colunas) em
  tabela pequena ou já filtrada por índice seletivo. Em tabela grande sem outro filtro, é o item 1.
- `OFFSET` profundo, `COUNT(*)` para existência, `DISTINCT` para esconder duplicata de JOIN,
  `ORDER BY RAND()`.
- Índice que provavelmente falta (FK, filtro obrigatório, filtro + ordenação da lista): diga qual.

## Não é achado

Subquery não correlacionada (`IN (SELECT ...)`, `EXISTS` com índice, escalar que roda uma vez), CTE
simples, estilo de SQL, "use cache/procedure/outro ORM" sem defeito concreto.

## Como escrever

Diga quantas linhas a query lê ou quantas vezes roda ("uma por pedido da página", "todas as notas da
empresa desde o início") e a correção em uma frase ("exija período de até 31 dias e pagine no servidor").
