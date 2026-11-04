# Injeção — SQL, NoSQL, comando, LDAP, XSS, SSRF

A raiz de toda injeção é a mesma: **dado controlado pelo usuário é interpretado como código/
comando** porque foi concatenado num contexto que mistura os dois. A defesa também é sempre a
mesma família: **separar dado de código** (parametrização/prepared statements) e, onde isso não
existir, **escapar para o contexto exato de saída**. Filtrar/sanitizar input é defesa secundária
e frágil isolada — a blocklist sempre tem furo.

Regra ao auditar: procure o ponto onde input do usuário entra numa string que será
*interpretada* (SQL, shell, HTML, template, path, header). Cada um desses é um contexto de
injeção distinto.

---

## 1. SQL Injection

### O que procurar
Concatenação de input em query. Os sinais, por stack:
- **PHP/ASP Classic:** `"SELECT ... WHERE id = " & request("id")`, `mysqli_query` com string
  montada, `.Execute("... '" & campo & "'")`. ASP Classic é território de risco alto porque
  raramente usa prepared statements por padrão.
- **PHP moderno / Laravel:** `DB::raw()`, `whereRaw()`, `->select(DB::raw("... $var"))` com
  variável interpolada; `mysqli`/PDO sem `prepare`.
- **Node/qualquer ORM:** `$queryRaw`/`$executeRaw` (Prisma), `.query("... " + x)`, template
  strings com input em `sequelize.query`.
- **Universal:** qualquer lugar onde o input aparece *dentro das aspas* da query em vez de vir
  como parâmetro ligado (`?`, `:nome`, `$1`).

### Critério de classificação
- **FAIL (crítico se rota pública):** input concatenado direto na query, sem parametrização nem
  cast. Confirme seguindo o dado da entrada até a query.
- **WARN:** query dinâmica onde parte é input mas há cast/validação parcial (ex.: `(int)$id`);
  ainda arriscado se a validação for incompleta ou houver outros parâmetros crus.
- **PASS:** prepared statements/binding em todo o caminho, ou ORM sem raw com input.

### Armadilhas que geram falso PASS
- **ORM não é imunidade.** `whereRaw`, `DB::raw`, `$queryRaw` e query builders com string
  interpolada injetam igual. ORM só protege quando o input vira *parâmetro ligado*.
- **Identificadores não parametrizam.** Nome de tabela/coluna/`ORDER BY` não aceita binding —
  aqui a defesa é **allowlist** de valores permitidos, não escape.
- **Cast a int resolve SÓ campos numéricos.** Não estenda a falsa sensação para strings.

### O que a correção vai exigir (aponte para hardening-seguranca-web)
Prepared statements/binding em 100% do caminho; allowlist para identificadores; nunca confiar em
escape manual como defesa principal.

---

## 2. NoSQL Injection

Menos óbvio que SQLi e frequentemente esquecido. Em bancos de documento (Mongo etc.), o input
não precisa quebrar aspas — basta chegar como **objeto/operador** em vez de string. Ex.: um login
que recebe `{ "user": {"$ne": null}, "pass": {"$ne": null} }` no lugar de strings pode autenticar
sem senha.

**Procure:** query montada a partir de `req.body`/`request` passado direto como filtro, sem
validar que os campos são strings/tipos esperados. **Defesa:** validação de tipo/schema estrita
antes da query; nunca passar body cru como filtro.

---

## 3. Injeção de comando (OS command)

O caso mais perigoso quando existe: input que chega a uma chamada de shell/exec.

**Procure:** `exec`, `system`, `shell_exec`, `passthru`, `proc_open`, `eval`, `WScript.Shell.Run`
(ASP), `child_process.exec` (Node), `Runtime.exec`, qualquer construção de comando com input.

**Critério:** input concatenado em comando de shell = **FAIL crítico** quase sempre. **Defesa:**
evitar shell; se inevitável, usar APIs que separam comando de argumentos (`execFile` com array de
args, não `exec` com string), e allowlist rígida. Nunca escape manual de shell como defesa
principal.

---

## 4. LDAP / XPath / outras injeções de sintaxe

Mesma família: input entra numa expressão de consulta (filtro LDAP, XPath, expressão de template
do lado servidor — SSTI). Procure montagem de filtro/expressão com input cru. Defesa:
escape específico da sintaxe + validação. SSTI (Server-Side Template Injection) é especialmente
perigoso porque frequentemente leva a execução remota de código — procure input do usuário
chegando a `render`/template engine como *template*, não como *dado*.

---

## 5. XSS (Cross-Site Scripting)

Injeção no contexto do **navegador**, não do servidor. Input do usuário é renderizado como HTML/
JS na página de outra vítima.

### Tipos
- **Refletido:** input volta na resposta imediata (busca, parâmetro de URL ecoado).
- **Armazenado:** input é salvo (comentário, perfil) e servido a outros — o mais grave, escala
  como injeção indireta.
- **DOM-based:** o JS do próprio front pega input (`location.hash`, query) e injeta no DOM.

### O que procurar
- Frameworks que auto-escapam (React, Vue, Angular templates) reduzem muito o risco — **mas**
  procure as saídas explícitas: `dangerouslySetInnerHTML`, `v-html`, `innerHTML`,
  `document.write`, `.html()` (jQuery), `Response.Write` em ASP com input.
- Templates que **não** auto-escapam (ASP Classic, EJS cru, PHP com `echo` direto) — todo output
  de input precisa de escape manual para o contexto (`htmlspecialchars`, `Server.HTMLEncode`).
- **Contexto importa:** escape para HTML ≠ escape para atributo ≠ escape para JS ≠ escape para
  URL. Escapar para o contexto errado não protege.

### Critério
- **FAIL:** input renderizado sem escape em contexto HTML/JS; `dangerouslySetInnerHTML`/`v-html`/
  `innerHTML` recebendo input não sanitizado.
- **WARN:** sanitização presente mas parcial, ou dependente só de CSP sem escape.
- **PASS:** auto-escape do framework em todo output de input, ou escape manual correto por
  contexto + CSP restritiva como defesa em profundidade.

### Defesa (aponte para hardening)
Escape por contexto de saída; `Content-Security-Policy` restritiva; sanitização de HTML (tipo
DOMPurify) quando HTML rico é realmente necessário; cookies de sessão `HttpOnly` para reduzir o
impacto de roubo de token via XSS.

---

## 6. Prototype pollution (ecossistema JS)

Específico de JS/Node. Merge/extend profundo de objeto controlado por usuário pode injetar
`__proto__`/`constructor`/`prototype` e corromper objetos globais, às vezes escalando para RCE.

**Procure:** deep merge/clone de `req.body` sem proteção; libs antigas de merge. **Defesa:**
rejeitar chaves `__proto__`/`constructor`/`prototype`; `Object.create(null)`; validação de schema.

---

## 7. SSRF (Server-Side Request Forgery)

O servidor faz uma requisição para um endereço **controlado pelo input do usuário**. O atacante
usa o servidor como proxy para alcançar a rede interna (metadata de cloud `169.254.169.254`,
serviços internos, localhost) ou para exfiltrar.

### O que procurar
Qualquer ponto onde input do usuário vira destino de uma requisição de saída: "importe esta URL",
webhooks configuráveis, preview de link, fetch de imagem por URL, integrações onde o endpoint é
parametrizável.

### Critério
- **FAIL:** input do usuário controla host/URL de requisição de saída sem allowlist.
- **WARN:** há validação de URL mas contornável (checa só o esquema, ou bloqueia por blocklist de
  IP que não cobre redirecionamento/DNS rebinding).
- **PASS:** allowlist de destinos + bloqueio de faixas internas + sem seguir redirect para fora
  da allowlist.

### Defesa
Allowlist de domínios/IPs de saída (egress) — a mesma lógica de egress da skill
`defesa-prompt-injection`. Bloquear faixas privadas e link-local. Não confiar em blocklist.

---

## Resumo — mapa de defesa por classe

| Classe | Defesa primária (arquitetural/determinística) | Defesa secundária (frágil isolada) |
|---|---|---|
| SQL/NoSQL | Prepared statements / binding; allowlist p/ identificadores | Escape manual, cast |
| Comando OS | Evitar shell; APIs com args separados; allowlist | Escape de shell |
| LDAP/XPath/SSTI | Separar dado de expressão; input nunca é template | Escape de sintaxe |
| XSS | Escape por contexto de saída; CSP | Sanitização/blocklist |
| Prototype pollution | Rejeitar chaves perigosas; schema | — |
| SSRF | Allowlist de egress; bloquear faixas internas | Blocklist de IP |
