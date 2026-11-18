# N+1 e query em loop

Uma query para buscar N registros e mais uma por registro. Com 20 linhas de teste ninguém vê; com 2 mil
na produção são 2.001 idas ao banco por requisição, e cada uma paga latência de rede.

## Onde procurar

- `for`, `forEach`, `map`, `while`, `reduce` com `await` de repositório, ORM ou `fetch` dentro.
- `Promise.all(lista.map(x => db...))`: não é N+1 sequencial, mas abre N queries ao mesmo tempo; sem teto
  esgota o pool de conexões. Conta como N+1.
- Serializer, presenter, DTO, template ou resolver de GraphQL que acessa relação (`pedido.cliente.nome`)
  com lazy loading.
- Getter/propriedade computada do model que consulta o banco.
- Validação por item: "confere se o produto existe" uma query por linha do arquivo importado.
- Gravação por item: `INSERT` ou `UPDATE` um por um em loop, em vez de lote.

## Por ORM: o sinal e a correção

| ORM | Sinal de N+1 | Correção |
|---|---|---|
| Prisma | `findUnique`/`findFirst` dentro de loop; relação lida depois sem `include` | `include`/`select` aninhado; ou `findMany({ where: { id: { in: ids } } })` e mapa em memória |
| TypeORM | relação `lazy`, ou `findOne` em loop | `relations`/`leftJoinAndSelect`; `In(ids)` |
| Sequelize | `getX()` da associação em loop | `include`; `where: { id: ids }` |
| Drizzle / Kysely / Knex | `select` por id em loop | `inArray`/`whereIn` com a lista; `JOIN` |
| Eloquent (Laravel) | `$pedido->cliente` em loop sem `with()` | `with('cliente')`, `load()`; `Model::preventLazyLoading()` em dev |
| Doctrine | proxy carregado em loop | `JOIN FETCH`; `addSelect` |
| Django | `obj.fk.campo` em loop | `select_related` (FK) e `prefetch_related` (N:N, reversa) |
| SQLAlchemy | lazy `relationship` em loop | `joinedload`/`selectinload` |
| ActiveRecord | associação em loop | `includes`/`preload`; gem `bullet` em dev |
| Hibernate/JPA | `FetchType.LAZY` em loop; `EAGER` em tudo também é ruim | `JOIN FETCH`, `@EntityGraph`, `@BatchSize` |
| Entity Framework | navegação lazy em loop | `Include`/`ThenInclude`; projeção com `Select` |
| Dapper / SQL cru | query dentro de loop | uma query com `IN`/`ANY(:ids)` ou `JOIN`; multi-mapping |

## Correções em ordem de preferência

1. Uma query com `JOIN` (relação 1:1 ou N:1) ou com a filha agregada (ver `subqueries.md`).
2. Duas queries: a lista, depois todos os filhos com `WHERE pai_id IN (:ids)`, e junção em memória por
   mapa. Limite o tamanho do `IN` (lotes de 500 a 1.000) para não estourar parâmetros.
3. Dataloader/batching no GraphQL.
4. Gravação em lote: `INSERT ... VALUES (...), (...)`, `createMany`, `bulk_create`, `COPY`, ou `UPDATE`
   com `CASE`/`FROM (VALUES ...)`.

Cuidado com o "eager loading de tudo": `include` de cinco relações numa lista sem limite troca o N+1 por
um produto cartesiano gigante. A lista de fora continua precisando de limite.

## Como escrever o achado

"N+1: `pedidos.map(p => db.cliente.findUnique({ where: { id: p.clienteId } }))` (arquivo:linha) faz uma
query por pedido; a lista tem 100 por página, então são 101 queries por chamada. Busque os clientes de uma
vez com `id: { in: ids }`."
