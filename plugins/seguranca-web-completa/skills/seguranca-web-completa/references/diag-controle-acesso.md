# Controle de acesso, autenticação e sessão

Controle de acesso quebrado é, ano após ano, a categoria nº 1 do OWASP. A razão é estrutural:
autenticação (quem é você) e autorização (o que você pode fazer) são verificações que precisam
acontecer **no servidor, em toda requisição**, e é fácil esquecer uma rota ou confiar no cliente.

Princípio-guia: **o servidor nunca confia no cliente.** Qualquer verificação que só existe no
frontend é decorativa — o atacante fala direto com a API.

---

## 1. Autenticação ausente ou incompleta

### O que procurar
- Rotas/endpoints que executam ação sensível sem exigir sessão/token válido.
- Middleware de auth aplicado em *algumas* rotas mas não em todas as protegidas — a rota
  esquecida é o furo.
- APIs internas/administrativas que assumem "ninguém conhece a URL" (segurança por obscuridade).
- Endpoints de debug/dev deixados em produção.

### Critério
- **FAIL:** rota que lê/muda dado sensível sem verificação de autenticação.
- **WARN:** auth presente mas cobertura inconsistente entre rotas equivalentes.
- **PASS:** toda rota não-pública exige auth verificada no servidor.

---

## 2. IDOR (Insecure Direct Object Reference)

O caso mais comum de acesso quebrado *dentro* de área autenticada. O usuário está logado, mas
consegue acessar recurso de **outro** usuário só trocando o id na requisição
(`GET /pedido/123` → `GET /pedido/124`).

### O que procurar
Toda rota que recebe um id/identificador de recurso e o usa para buscar/alterar **sem verificar
que o recurso pertence ao usuário autenticado** (ou que ele tem permissão sobre ele). O padrão
falho: `SELECT * FROM pedido WHERE id = :id` sem `AND user_id = :usuarioLogado`.

### Critério
- **FAIL:** recurso acessível por id sem checagem de posse/permissão. Confirme trocando o id.
- **WARN:** checagem presente em alguns endpoints do recurso mas não em todos (ex.: `GET` checa,
  `DELETE` não).
- **PASS:** toda operação sobre recurso valida posse/permissão no servidor.

**Nota:** UUID em vez de id sequencial **reduz** a facilidade (não dá pra enumerar `+1`), mas
**não é a defesa** — UUID vaza em logs, URLs, respostas de API. A defesa é a checagem de posse.
Aponte isso; não deixe passar "usamos UUID, então estamos protegidos".

---

## 3. RBAC / escalação de privilégio

### O que procurar
- Ações de admin (criar/editar/excluir, aprovar, resetar senha de outro) sem verificação de
  **papel/permissão**, só de autenticação. Estar logado ≠ ser admin.
- Verificação de papel feita no frontend (esconder o botão) mas não no backend (o endpoint aceita
  a chamada de qualquer um).
- Campos de privilégio alteráveis pelo usuário: `role`, `is_admin`, `permissions` chegando no
  `body` de um update de perfil e sendo persistidos (mass assignment) → o usuário se promove.

### Critério
- **FAIL:** ação privilegiada sem checagem de papel no servidor; ou campo de privilégio
  atribuível via input.
- **WARN:** checagem de papel presente mas granularidade insuficiente (tudo-ou-nada onde
  deveria haver permissão por módulo).
- **PASS:** cada ação privilegiada verifica papel/permissão específica no servidor; campos de
  privilégio não são atribuíveis por input do usuário (allowlist de campos no update).

---

## 4. Trust no frontend (a falha que amarra as anteriores)

### O que procurar
- Frontend guarda papel/permissões em `localStorage` e decide acesso com base nisso →
  manipulável pelo devtools para forjar admin.
- Rotas de admin no SPA protegidas só por guard de roteamento cliente, sem o backend revalidar.
- Menu/botões escondidos por permissão, mas os endpoints correspondentes aceitam a chamada.

### Critério
- **FAIL:** decisão de autorização tomada no cliente sem revalidação no servidor.
- **PASS:** frontend só *exibe*; toda autorização é decidida e imposta no servidor; no load, o
  frontend valida a sessão contra o servidor (ex.: `GET /me`) em vez de confiar no storage local.

---

## 5. Autenticação — robustez do login

### O que procurar e critério

**a) Rate limit / brute force.**
Login sem limite de tentativas = brute force / credential stuffing livre.
- **FAIL:** endpoint de login sem rate limit.
- **PASS:** limite por IP+conta (padrão comum: ~5 tentativas / 15 min), com backoff.
- Rate limit sozinho não impede credential stuffing distribuído — combine com detecção de
  anomalia e MFA nos casos sensíveis.

**b) Lockout / detecção de falhas repetidas.**
Rastrear falhas por conta; considerar bloqueio temporário ou MFA obrigatório após N falhas.

**c) Política de senha.**
Comprimento mínimo razoável (o consenso atual — NIST 800-63B — prioriza **comprimento** e checagem
contra listas de senhas vazadas, e **desencoraja** regras de complexidade obrigatória e troca
periódica forçada). Não recomende "trocar senha a cada 90 dias" nem "1 maiúscula + 1 símbolo" como
se fossem best practice atual — não são mais.

**d) Enumeração de usuário.**
Mensagem de erro que diferencia "usuário não existe" de "senha errada" permite enumerar contas.
Resposta genérica ("credenciais inválidas") + timing constante.

**e) Recuperação de senha.**
Token de reset com expiração curta, uso único, entropia alta; rate limit no envio; não vazar se o
e-mail existe.

---

## 6. Sessão e tokens

### O que procurar e critério

**a) Onde o token vive (frontend).**
Token em `localStorage` é acessível por qualquer XSS. Cookie `HttpOnly`+`Secure`+`SameSite` não é
lido por JS.
- **WARN/FAIL** (conforme superfície de XSS): token de sessão em `localStorage`.
- **PASS:** sessão em cookie `HttpOnly`, `Secure`, `SameSite=Strict/Lax`.

**b) Invalidação.**
- O logout realmente invalida a sessão no servidor, ou só apaga o token do cliente?
- JWT stateless: um token válido continua válido até expirar mesmo após logout/desativação do
  usuário — a menos que haja blacklist ou checagem de estado no servidor.
- **Procure:** middleware que só valida a assinatura do JWT sem verificar se o usuário ainda
  existe/está ativo. Usuário demitido com token válido = acesso mantido.
- **PASS:** revogação efetiva (sessão server-side, ou blacklist/versão de token, ou checagem de
  estado do usuário com cache curto).

**c) Config do JWT.**
- Segredo forte, vindo de variável de ambiente, nunca hardcoded.
- Algoritmo **restrito** (rejeitar `alg: none`; fixar o algoritmo esperado — não deixar o token
  escolher).
- Expiração razoável; refresh token com segredo/rotação separados.

**d) Fixation / CSRF.**
- Regenerar id de sessão no login (anti session fixation).
- Ações que mudam estado via cookie de sessão precisam de proteção **CSRF** (token anti-CSRF ou
  `SameSite`); APIs com token em header (Authorization) são naturalmente menos expostas a CSRF.

---

## Resumo

| Falha | Sinal | Defesa |
|---|---|---|
| Auth ausente | Rota sensível sem verificação | Auth no servidor em toda rota não-pública |
| IDOR | Id de recurso sem checar posse | Validar posse/permissão no servidor (UUID não basta) |
| Escalação/RBAC | Ação de admin só com auth; mass assignment | Checar papel no servidor; allowlist de campos |
| Trust no frontend | Autorização no cliente | Servidor decide e impõe; frontend só exibe |
| Brute force | Login sem rate limit | Rate limit + lockout + MFA sensível |
| Sessão frágil | Token em localStorage; logout não invalida | Cookie HttpOnly; revogação efetiva; JWT com alg fixo |
