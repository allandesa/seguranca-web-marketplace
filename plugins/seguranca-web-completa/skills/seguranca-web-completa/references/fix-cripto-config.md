# Correção — criptografia, segredos, config, headers, logging, deps, LGPD

Correções de postura. Padrão + critério. Cross-stack; boa parte se resolve na config do servidor,
independente da linguagem.

---

## 1. Cifra de PII em repouso

**Padrão:** cifra autenticada para os campos sensíveis que o modelo de ameaça exige, chave fora
do código.
```
// AES-256-GCM (autenticada: garante confidencialidade E integridade)
cifrado = aesGcmEncrypt(dado, chave, iv)     // iv único por operação
// chave vem de env/cofre (KMS/Vault), NUNCA do código nem do mesmo repositório
```
Não cifre o que não precisa (cifrar tudo complica busca/índice sem ganho). Cifre o que dói vazar:
CPF/CNPJ, dado de saúde/financeiro, tokens.

**Critério:** campos sensíveis cifrados com AES-GCM (ou equivalente autenticado); chave em
cofre/env; IV único por registro.

---

## 2. Gestão de segredos → env/cofre + rotação

**Padrão:**
- Segredos (chave de API, senha de banco, segredo de JWT, credencial de cloud) em **variáveis de
  ambiente** ou **cofre** (KMS/Vault/Secrets Manager). Nunca no código.
- `.env` no `.gitignore`; `.env.example` só com placeholders.
- **Rotação:** qualquer segredo que esteve no código ou no histórico do git está **comprometido**.
  A correção é **rotacionar** (gerar novo, invalidar antigo), não só remover o arquivo — o
  histórico do git guarda o valor antigo.
- Varra o repositório e o histórico com TruffleHog/Gitleaks para achar segredos esquecidos.

**Critério:** nenhum segredo no código/histórico ativo; os que vazaram foram rotacionados; scan
limpo.

---

## 3. Transporte → HTTPS forçado + HSTS

**Padrão:** redirect HTTP→HTTPS; HSTS; cookies `Secure`; TLS moderno.
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```
Cookies de sessão: `Secure; HttpOnly; SameSite=Strict` (ou `Lax` conforme fluxo).

**Critério:** HTTP redireciona; HSTS ativo; cookies com atributos seguros.

---

## 4. Cabeçalhos de segurança HTTP

Setar todos (via middleware tipo Helmet no Node, **ou** direto na config do servidor —
IIS/Apache/Nginx — para stacks sem middleware):

```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'none'; frame-ancestors 'none'
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```
`frame-ancestors 'none'` (ou `X-Frame-Options: DENY`) contra clickjacking. Evite `'unsafe-inline'`
em `script-src`.

**Critério:** headers presentes na resposta; CSP sem `unsafe-inline` em script; clickjacking
bloqueado.

---

## 5. CORS restrito

**Padrão:** allowlist de origens vinda de config/env. Nunca `*` junto com credenciais; nunca
refletir a origem recebida sem validar.
```
const origensPermitidas = new Set(env.CORS_ORIGINS.split(","));
if (origensPermitidas.has(origemRecebida)) permitir(origemRecebida);   // eco só do que está na lista
// credentials: true SOMENTE com origem específica, nunca com '*'
```

**Critério:** CORS não é wildcard com credenciais; só origens da allowlist passam.

---

## 6. Erros → genérico ao cliente, detalhe no log

**Padrão:** em produção, resposta de erro genérica ao cliente (sem stack trace, query, path,
versão). Detalhe completo só no log server-side, correlacionado por id de requisição.

No contexto **ASP Classic/IIS**: página de erro customizada com `responseMode` adequado; nunca
`existingResponse` que vaze o erro detalhado ao cliente; log do detalhe via Event Log/arquivo,
não na tela.

**Critério:** erro de produção não vaza interno; detalhe existe só no log.

---

## 7. Logging com máscara de PII

**Padrão dual:**
- **Logar** eventos de segurança: login (sucesso/falha) com IP e timestamp, 403 (acesso negado)
  com recurso pedido, rate limit (429), ações administrativas, modificações de dado sensível.
- **Mascarar** PII/segredo no log: senha, token, CPF, e-mail nunca em texto. Máscara na origem.
```
log.info("login_falha", { ip, userId: mascarar(email), quando });   // não logue a senha, nunca
```
Proteger integridade (a aplicação não apaga/edita o próprio log), rotação, e centralização quando
possível.

**Critério:** eventos de segurança presentes no log com contexto; nenhum segredo/PII em texto no
log; log protegido e rotado.

---

## 8. Dependências e componentes

**Padrão:**
- Rodar audit no CI (`npm audit` / `composer audit` / equivalente); falhar o build em CVE
  crítica/alta explorável.
- Commitar o **lock file** (build reproduzível).
- Atualizar/remover deps vulneráveis; runtime dentro do suporte.

**Legado em EOL (ASP Classic / Zend descontinuado):** não há patch. Mitigação por **camada** —
WAF/regras no servidor, isolamento de rede, egress restrito — e priorizar a migração dos módulos
mais expostos. Documente como risco residual assumido, não como "resolvido".

**Critério:** audit no CI; lock versionado; deps sem CVE crítica no caminho usado; legado em EOL
documentado com controle compensatório.

---

## 9. Checklist LGPD (quando processa dado pessoal)

Correção de conformidade, complementar à de segurança:
- **Direito de acesso:** endpoint/processo de exportação dos dados do titular.
- **Direito de eliminação:** exclusão efetiva (não só soft delete quando o titular exige apagar).
- **Anonimização:** processo para anonimizar quando aplicável.
- **Consentimento:** coleta e revogação registráveis.
- **PII cifrada em repouso** (item 1).
- **Incidente:** exposição de dado pessoal por falha de segurança é reportável (ANPD/titular) —
  tenha o processo de resposta definido.

**Critério:** direitos do titular implementados; PII cifrada; processo de incidente definido.

---

## Resumo

| Classe | Correção-raiz | Critério |
|---|---|---|
| PII repouso | AES-GCM + chave em cofre | Sensível cifrado, chave fora do código |
| Segredos | Env/cofre + rotação do que vazou | Sem segredo no histórico; rotacionado |
| Transporte | HTTPS+HSTS+cookies Secure | HTTP redireciona; HSTS ativo |
| Headers | CSP/HSTS/nosniff/frame | Headers presentes; CSP sem unsafe-inline |
| CORS | Allowlist de origem | Sem wildcard com credenciais |
| Erros | Genérico ao cliente | Sem stack trace em prod |
| Logging | Logar segurança, mascarar PII | Eventos logados; PII mascarada |
| Deps | Audit no CI; compensar EOL | Sem CVE crítica; legado documentado |
| LGPD | Direitos do titular + cifra | Acesso/eliminação/consentimento ativos |
