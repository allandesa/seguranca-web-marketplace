# Escrita, transação e migration

## UPDATE e DELETE

- Sem `WHERE`, ou com `WHERE` montado de filtro opcional que pode chegar vazio: **bloqueia**. O ORM
  também faz isso: `updateMany({})`, `deleteMany({})`, `Model::query()->update(...)`, `.objects.all().delete()`.
- Em massa numa transação só (centenas de milhares de linhas): trava a tabela ou as linhas por minutos,
  enche o log de transação e a replicação atrasa. Faça em lotes (`LIMIT`/chave por faixa) com commit por
  lote, de preferência em job.
- Apagar dado de tabela que outra tabela referencia sem índice na FK: cada linha apagada faz scan na
  filha.
- Soft delete (`deletado_em`) sem o filtro em toda leitura, ou sem índice que o considere.

## Transação

- Transação aberta enquanto chama API externa, envia e-mail, gera PDF ou espera o usuário: segura lock e
  conexão do pool o tempo todo. Feche antes, ou use outbox.
- `SELECT ... FOR UPDATE` sem índice no filtro: trava mais linhas que as necessárias (no MySQL/InnoDB,
  trava faixas do índice percorrido).
- Ordem de lock diferente em dois caminhos de código: deadlock sob carga.
- Leitura e escrita "ler, calcular, gravar" sem lock nem versão: atualização perdida em concorrência
  (saldo, estoque, numeração). Use `UPDATE ... SET x = x - :n WHERE x >= :n` atômico, lock pessimista ou
  coluna de versão.
- Numeração sequencial (número de nota, pedido) por `MAX(numero) + 1`: duplica em concorrência. Use
  sequence ou tabela de numeração com lock.

## Migration em tabela grande

| Operação | Risco | Forma segura (varia por banco) |
|---|---|---|
| `CREATE INDEX` | trava escrita na tabela durante a criação | PostgreSQL: `CREATE INDEX CONCURRENTLY` (fora de transação); MySQL 8: `ALGORITHM=INPLACE, LOCK=NONE`; SQL Server: `ONLINE = ON` |
| Coluna nova `NOT NULL` com `DEFAULT` | em versões antigas reescreve a tabela inteira | adicionar nula, preencher em lotes, depois `NOT NULL`; PostgreSQL 11+ faz `DEFAULT` constante sem reescrever, mas não `DEFAULT` volátil |
| Trocar tipo de coluna | reescreve a tabela e os índices | coluna nova, cópia em lotes, troca na aplicação, remoção depois |
| `ADD FOREIGN KEY` / `CHECK` | valida a tabela inteira com lock | PostgreSQL: `NOT VALID` e depois `VALIDATE CONSTRAINT` |
| Backfill (`UPDATE` de todas as linhas) na mesma migration | transação gigante, lock longo | job em lotes, separado da migration de esquema |
| `DROP COLUMN`/`RENAME` de coluna ainda lida | quebra a versão em produção durante o deploy | expandir e contrair: a aplicação para de usar, depois a migration remove |
| Migration que não roda duas vezes | quebra ao reexecutar ou em ambiente parcial | `IF NOT EXISTS`/`IF EXISTS`, ou controle de versão que garante uma vez só |

Pergunte sempre: quantas linhas essa tabela tem em produção, e a migration roda com a aplicação no ar?
