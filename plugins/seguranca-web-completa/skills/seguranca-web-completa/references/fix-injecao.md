# Correção — injeção (SQLi, XSS, comando, SSRF, prototype pollution)

Padrão + critério de aceitação por classe. Os exemplos são ilustrativos e cross-stack; adapte a
sintaxe à linguagem do projeto. O que **não** muda entre stacks é o princípio.

---

## 1. SQL Injection → parametrização

**Padrão:** o input do usuário nunca entra na string da query. Ele vai como **parâmetro ligado**;
o driver mantém dado e código separados.

Ilustrativo (o conceito, não o framework):
```
// ERRADO — concatenação
query("SELECT * FROM user WHERE email = '" + email + "'")

// CERTO — parâmetro ligado (a sintaxe do placeholder varia por driver)
query("SELECT * FROM user WHERE email = ?", [email])            // ? posicional
query("SELECT * FROM user WHERE email = :email", { email })     // nomeado
```

**Legado sem framework (ASP Classic / PHP antigo):** use os prepared statements do próprio driver
(ADO `Command` com `Parameters.Append` no ASP; `mysqli`/PDO `prepare`+`bind` no PHP). Não existe
"helper mágico" — existe o mecanismo cru do driver, e ele resolve.

**Identificadores (tabela/coluna/ORDER BY)** não aceitam binding. Defesa: **allowlist**.
```
const colunasPermitidas = { nome: "nome", data: "created_at" };
const coluna = colunasPermitidas[input] ?? "created_at";   // input só escolhe da lista
```

**Critério de aceitação:** 100% das queries com input usam binding; identificadores dinâmicos vêm
de allowlist; nenhuma concatenação de input em SQL sobrou. Confirme com o teste de SQLi da
auditoria (payload não altera o resultado).

---

## 2. XSS → escape por contexto de saída + CSP

**Padrão:** todo dado de origem não-confiável é escapado **para o contexto exato** onde é
renderizado. Escape de HTML ≠ atributo ≠ JS ≠ URL.

- **Framework com auto-escape (React/Vue/Blade/Angular):** deixe auto-escapar. **Remova** os
  bypasses onde há input não sanitizado: `dangerouslySetInnerHTML`, `v-html`, `innerHTML`,
  `.html()`. Se HTML rico é mesmo necessário, sanitize com allowlist (tipo DOMPurify) antes.
- **Template sem auto-escape (ASP Classic/PHP cru/EJS):** escape manual em todo output de input —
  `Server.HTMLEncode` (ASP), `htmlspecialchars($x, ENT_QUOTES)` (PHP). Escape no contexto certo:
  dentro de atributo, dentro de `<script>`, dentro de URL — cada um tem sua função.

**CSP como segunda camada:**
```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'none'
```
Evite `'unsafe-inline'` em `script-src` (anula boa parte do valor). Nonce/hash para scripts
inline necessários.

**Reduzir impacto:** cookie de sessão `HttpOnly` — assim, mesmo um XSS não rouba o token de
sessão via JS.

**Critério:** nenhum output de input sem escape de contexto; bypasses de auto-escape só sobre
conteúdo sanitizado; CSP restritiva ativa. Confirme com o payload marcador da auditoria (não
executa).

---

## 3. Injeção de comando → evitar shell / args separados

**Padrão:** não construa comando de shell com input. Se precisa chamar um programa externo, use a
API que passa **argumentos como array**, sem shell interpretando a string.
```
// ERRADO
exec("convert " + arquivo + " saida.png")          // shell interpreta input

// CERTO
execFile("convert", [arquivo, "saida.png"])         // args separados, sem shell
```
Se shell for inevitável, allowlist rígida dos valores possíveis. `eval`/execução dinâmica de
código com input: remover.

**Critério:** nenhuma construção de comando concatena input; chamadas externas usam args
separados; sem `eval` sobre input.

---

## 4. SSRF → allowlist de egress

**Padrão:** requisição de saída cujo destino depende de input só alcança **destinos aprovados**.
```
const egressPermitido = new Set(["api.parceiro.com", "cdn.meudominio.com"]);
const url = new URL(inputUsuario);
if (!egressPermitido.has(url.hostname)) reject("destino não permitido");
// bloquear também faixas internas: 127.0.0.0/8, 10/8, 172.16/12, 192.168/16, 169.254.169.254
// e NÃO seguir redirect para fora da allowlist
```
Mesma lógica de egress da skill `defesa-prompt-injection` — se o sistema tem IA e faz requisições,
o egress vale para os dois.

**Critério:** destino de saída restrito por allowlist; faixas internas/link-local bloqueadas;
redirect não escapa da allowlist.

---

## 5. Prototype pollution (Node/JS) → rejeitar chaves perigosas + schema

**Padrão:** valide o corpo contra um schema (só as chaves esperadas passam) e/ou rejeite
`__proto__`/`constructor`/`prototype` antes de qualquer merge profundo.
```
if (["__proto__","constructor","prototype"].some(k => k in bodyRecebido)) reject();
// melhor: validar contra schema com allowlist de campos, descartando o resto
```

**Critério:** merge/clone de objeto de input protegido; validação de schema com allowlist de
campos; chaves perigosas rejeitadas.

---

## Nota transversal: validação de input

Validar/sanitizar input (tipo, formato, tamanho, allowlist de valores) é **defesa em
profundidade** valiosa — reduz superfície e pega erro cedo. Mas nunca é a defesa **principal**
contra injeção: a principal é separar dado de código (parametrizar, escapar por contexto, args
separados). Entregue validação **junto** com a defesa estrutural, nunca **no lugar** dela.
