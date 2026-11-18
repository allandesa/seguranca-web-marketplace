# Volume e filtros: nada de "todos os registros da vida"

A falha mais comum e mais cara não tem nada de sofisticado: uma leitura sem teto numa tabela que cresce
todo dia. Ela passa no teste (base pequena), passa na homologação e cai em produção no mês em que a
tabela fica grande, ou no dia em que alguém clica em "Exportar" sem filtro.

## Toda leitura de tabela que cresce precisa de três coisas

1. **Filtro obrigatório e seletivo**, validado no servidor. Para dado temporal (nota, venda, movimento,
   log, evento, mensagem, pedido): período obrigatório, com teto (ex.: até 31 ou 92 dias por consulta).
2. **Limite** de linhas devolvidas: paginação, `LIMIT/TOP/FETCH FIRST`, `take`, `limit`, `paginate()`.
   O servidor impõe um máximo (ex.: `pageSize <= 200`) mesmo que o cliente peça mais.
3. **Ordem estável** (chave única no fim do `ORDER BY`), senão a paginação repete e pula linha.

Faltou qualquer uma: achado.

## Como o "todos" escapa

| Padrão | Exemplo | Por que quebra |
|---|---|---|
| Filtro opcional que chega vazio | `if (filtro.cliente) where.clienteId = filtro.cliente` sem nenhum filtro obrigatório | sem parâmetro vira a tabela inteira |
| Período sem teto | `dataIni` obrigatória, `dataFim` opcional; ou "desde 2000" como padrão | "todas as notas da vida" |
| Valor padrão amplo | `dataIni = '1900-01-01'`, `status IN (todos)`, `filial = '%'` | o padrão é o pior caso |
| `LIKE '%' || :termo || '%'` com termo vazio | busca sem texto | casa com tudo, e sem índice |
| Paginação só no front | API devolve array completo; a tabela da tela fatia | trafega e serializa tudo |
| "Exportar" e "Imprimir" | reaproveita a query da tela sem o limite | o limite some justamente na operação mais pesada |
| Dashboard/indicador | `SUM`/`COUNT` sobre a tabela inteira a cada abertura de tela | scan completo a cada F5 |
| Job/cron | `SELECT * FROM fila WHERE processado = false` sem `LIMIT` | depois de uma falha, a fila acumula e o job estoura memória |
| `COUNT(*)` total para paginação | conta a tabela filtrada inteira a cada página | caro em tabela grande; considere "tem mais?" com `LIMIT n+1` |

## Combo, select e dropdown de cadastro

Filtro de tela "Cliente" como `<select>` que carrega **todos os clientes** (ou produtos, fornecedores,
colaboradores, notas) é o mesmo erro do lado da tela:

- a API de opções faz `SELECT id, nome FROM cliente` sem filtro;
- o navegador monta milhares de `<option>`;
- a tela abre lenta e trava com o crescimento do cadastro.

O certo:

- **busca no servidor** (autocomplete): mínimo de 2 ou 3 caracteres, `LIMIT` (ex.: 20), ordenado por
  relevância ou nome, com índice que sirva ao predicado (prefixo `nome LIKE 'abc%'`, trigram/full-text
  para "contém");
- debounce no front e cancelamento da requisição anterior;
- o valor já selecionado carregado por id, não pela lista inteira;
- lista completa só para domínio pequeno e fechado (UF, status, tipo), de preferência em cache.

Achado típico: "O filtro de Cliente carrega o cadastro inteiro: `GET /clientes/opcoes` faz
`findMany({ select: { id, nome } })` sem `where` nem `take`. Troque por busca no servidor com mínimo de
3 caracteres e `take: 20`."

## Exportação, relatório e "fiscal"

- Período obrigatório e com teto; acima do teto, a exportação vira job assíncrono que gera arquivo, não
  uma requisição que segura a conexão.
- Leitura em blocos (cursor/stream/keyset), nunca o resultado inteiro em memória.
- Agregação no banco (`GROUP BY`), não no código depois de trazer as linhas.
- Documento fiscal, financeiro e de auditoria cresce sem parar e nunca é apagado: trate sempre como
  tabela grande.

## Paginação

- `OFFSET` alto lê e descarta todas as linhas anteriores: página 5.000 de 50 lê 250 mil linhas. Para
  lista longa ou rolagem infinita, use keyset: `WHERE (data, id) < (:ultima_data, :ultimo_id) ORDER BY
  data DESC, id DESC LIMIT 50`, com índice `(data, id)`.
- `OFFSET` é aceitável quando a navegação real não passa de poucas páginas.

## Perguntas para o revisor

- Se eu chamar este endpoint sem nenhum parâmetro, o que volta?
- E com o período máximo permitido, quantas linhas, em 5 anos de dado?
- O limite está no servidor? O que acontece com `?pageSize=100000`?
- A exportação usa a mesma query sem o limite?
- O combo de filtro carrega o cadastro inteiro?
