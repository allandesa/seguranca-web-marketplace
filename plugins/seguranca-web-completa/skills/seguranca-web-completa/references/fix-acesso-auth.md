# Correção — controle de acesso, autenticação e sessão

Padrão + critério por classe. O princípio invariante: **o servidor decide e impõe toda
autorização; o cliente só exibe.**

---

## 1. Auth ausente → verificação no servidor em toda rota não-pública

**Padrão:** middleware/filtro de autenticação aplicado por padrão, com as rotas públicas sendo a
**exceção explícita** (allowlist de rotas públicas), não o contrário. Assim, esquecer de proteger
uma rota falha para o lado seguro (fica protegida) em vez de aberta.
```
// aplicar auth globalmente; liberar só o que é público explicitamente
app.usarAuth(exceto = ["/login", "/health", "/public/*"]);
```
Endpoints de debug/dev: remover de produção.

**Critério:** toda rota que lê/muda dado sensível exige sessão/token válido, verificado no
servidor.

---

## 2. IDOR → verificação de posse/permissão no servidor

**Padrão:** toda busca/alteração por id inclui a condição de que o recurso pertence ao usuário
autenticado (ou que ele tem permissão sobre ele). A verificação é no **servidor**, na query ou
logo após.
```
// ERRADO: confia no id sozinho
SELECT * FROM pedido WHERE id = :id

// CERTO: amarra ao usuário autenticado
SELECT * FROM pedido WHERE id = :id AND user_id = :usuarioLogado
// ou, se admin pode ver tudo: checar papel antes de liberar o bypass do filtro
```
Aplique a **todos os verbos** do recurso (GET, PUT, DELETE) — é comum proteger o GET e esquecer o
DELETE.

**UUID não é a defesa.** Ajuda contra enumeração, mas a defesa é a checagem de posse. Não
substitua uma pela outra.

**Critério:** trocar o id na requisição não dá acesso a recurso de outro usuário, em nenhum verbo.

---

## 3. Escalação / RBAC → papel verificado no servidor + anti mass assignment

**Padrão A — checagem de papel/permissão por ação, no servidor.**
```
// cada ação privilegiada verifica a permissão específica
if (!usuario.pode("pedidos", "aprovar")) reject(403);
```
Granularidade por módulo/ação quando o negócio exige, não só "admin sim/não".

**Padrão B — bloquear mass assignment.**
Campos de privilégio (`role`, `is_admin`, `permissions`) **não** podem ser atribuídos por input.
Use allowlist de campos no update; descarte o resto.
```
const camposPermitidos = ["nome", "email", "telefone"];   // role NÃO está aqui
const dados = pick(bodyRecebido, camposPermitidos);
```

**Critério:** ação privilegiada exige papel verificado no servidor; usuário não consegue se
promover via input.

---

## 4. Trust no frontend → servidor impõe, frontend valida sessão

**Padrão:**
- Autorização é **decidida e imposta no servidor**. O frontend esconder um botão é UX, não
  segurança — o endpoint correspondente também verifica.
- No carregamento, o frontend **valida a sessão contra o servidor** (ex.: `GET /me`) e usa os
  dados do servidor para papel/permissões, **não** o `localStorage` (manipulável).
- Se o servidor responde 401/conta desativada, logout automático.

**Critério:** manipular `localStorage` não concede acesso; todo endpoint privilegiado rejeita
chamada sem o papel correto, independentemente do que o frontend mostra.

---

## 5. Hashing de senha → bcrypt / scrypt / Argon2

**Padrão:** algoritmo lento, específico para senha, com salt por senha (as libs cuidam do salt).
```
// bcrypt com custo adequado (≥ 10-12), ou Argon2id (recomendação atual mais forte)
hash = bcrypt(senha, custo=12)
confere = bcrypt.verificar(senhaInformada, hashArmazenado)
```
**Não** use MD5/SHA1/SHA-256 para senha — rápidos demais, feitos para dados, não para senha. Se o
banco tem hashes fracos, migre no próximo login bem-sucedido (rehash) e force reset onde possível.

**Critério:** senhas em bcrypt/scrypt/Argon2 com custo adequado; nenhum hash rápido nem plaintext.

---

## 6. Sessão e JWT seguros

**a) Onde guardar (frontend):** sessão em cookie `HttpOnly` + `Secure` + `SameSite=Strict/Lax`.
Isso tira o token do alcance de XSS. Evite `localStorage` para token de sessão.

**b) Invalidação efetiva:**
- Sessão server-side: logout apaga a sessão no servidor.
- JWT stateless: para revogar antes da expiração, use **blacklist**, **versão de token** (campo
  incrementável no usuário que invalida tokens antigos), ou **checagem de estado do usuário** a
  cada requisição (com cache curto, ex.: 1-2 min) — assim usuário desativado perde acesso mesmo
  com token ainda válido.
```
// no middleware de auth, além de validar a assinatura:
const u = verificarUsuarioAtivo(userId);   // cache curto
if (!u.existe || !u.ativo) reject(401);
```

**c) Config do JWT:**
- Segredo forte via env, nunca hardcoded.
- **Fixar o algoritmo** esperado na verificação (rejeitar `alg: none` e troca de algoritmo).
- Expiração curta no access token; refresh com segredo/rotação separados.

**d) Anti-fixation e CSRF:**
- Regenerar id de sessão no login.
- Ações que mudam estado via cookie precisam de token anti-CSRF ou `SameSite`. APIs com token em
  header `Authorization` são naturalmente menos expostas a CSRF.

**Critério:** logout/desativação invalidam acesso efetivamente; `alg` fixo; token de sessão fora
do localStorage; ações sensíveis protegidas contra CSRF.

---

## 7. Rate limit e brute force → limite no login + anomalia

**Padrão:**
- Login: limite por IP **e** por conta (padrão comum ~5/15min), com backoff crescente.
- Reset de senha: limite por conta/IP.
- API geral: cota por usuário/janela.
- Resposta genérica no login ("credenciais inválidas") + timing constante para não permitir
  enumeração de usuário.
- MFA para contas sensíveis; considerar bloqueio temporário/desafio após N falhas.

Rate limit sozinho não impede credential stuffing distribuído — combine com detecção de anomalia
e, idealmente, MFA.

**Sobre política de senha (atualizando prática antiga):** o consenso atual (NIST 800-63B) prioriza
**comprimento mínimo** e checagem contra listas de senhas vazadas, e **não** recomenda mais regras
de complexidade obrigatória nem troca periódica forçada. Não implemente "trocar a cada 90 dias" ou
"exigir 1 símbolo + 1 maiúscula" achando que é best practice — a orientação mudou.

**Critério:** login e reset com rate limit; sem enumeração de usuário; MFA disponível para
sensível.

---

## Resumo

| Falha | Correção-raiz | Critério |
|---|---|---|
| Auth ausente | Auth global, público como exceção | Rota sensível exige token válido |
| IDOR | Checar posse/permissão no servidor | Trocar id não vaza recurso alheio |
| Escalação | Papel no servidor + allowlist de campos | Não se auto-promove; ação exige papel |
| Trust frontend | Servidor impõe; frontend valida sessão | localStorage manipulado não dá acesso |
| Senha | bcrypt/scrypt/Argon2 | Sem hash rápido/plaintext |
| Sessão/JWT | HttpOnly; revogação; alg fixo | Logout/desativação cortam acesso |
| Brute force | Rate limit login+reset; MFA | Sem brute force livre nem enumeração |
