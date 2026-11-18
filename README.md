# allan-skills

Skills e plugins do Allan Carvalho.

```
/plugin marketplace add allandesa/seguranca-web-marketplace
/plugin install seguranca-web-completa@allan-skills
/plugin install revisao-banco@allan-skills
```

| Plugin | Skills | Para quê |
|---|---|---|
| `seguranca-web-completa` | `seguranca-web-completa` | Auditoria de segurança web/API (OWASP, IDOR, injeção, upload, segredos, prompt injection, LGPD), read-only por padrão |
| `revisao-banco` | `revisao-queries`, `plano-de-execucao` | Revisão de queries no código (leitura sem teto, combo de cadastro inteiro, subquery por linha, N+1, índice, escrita em massa, migration) e medição no banco (EXPLAIN, pg_stat_statements, slow log) |

`revisao-queries` traz `checklist.md`, a versão curta para revisor automático de PR.
