# Criptografia, configuração, headers, logging e dependências

Reúne quatro classes do OWASP que compartilham a mesma natureza: falhas de **postura**, não de
lógica de código. Raramente há um "bug" localizável; o problema é uma configuração ausente,
fraca ou default.

---

## 1. Falhas de criptografia (OWASP A02)

### a) Hashing de senha
- **Procure:** senha em texto plano no banco (FAIL crítico imediato); hash com algoritmo
  quebrado (MD5, SHA1 puro); hash sem salt.
- **PASS:** algoritmo lento e específico para senha — **bcrypt, scrypt ou Argon2** — com custo/
  rounds adequados (bcrypt ≥ 10-12; Argon2 é a recomendação atual mais forte). SHA-256 "com salt"
  **não** é suficiente para senha — é rápido demais, feito para hash de dados, não de senha.

### b) Dados sensíveis / PII em repouso
- **Procure:** PII (CPF, CNPJ, dados de saúde, financeiros), tokens e segredos armazenados sem
  cifra quando o modelo de ameaça exige (host compartilhado, backup exposto, requisito LGPD).
- **PASS:** cifra autenticada (ex.: AES-256-GCM) para os campos que precisam, com a chave em
  variável de ambiente / cofre (KMS/Vault), **nunca no código nem no mesmo repositório**.
- Contexto BR/LGPD: exposição de dado pessoal por falha de cripto é incidente reportável.

### c) Dados em trânsito
- **Procure:** HTTP sem redirect para HTTPS; TLS desatualizado; ausência de HSTS; cookies sem
  `Secure`.
- **PASS:** HTTPS forçado (redirect + HSTS com `max-age` longo), TLS moderno, cookies `Secure`.

### d) Aleatoriedade
- **Procure:** `rand()`/`Math.random()` gerando token/segredo/senha temporária — previsível.
- **PASS:** gerador criptograficamente seguro (CSPRNG) para qualquer valor de segurança.

---

## 2. Segredos e configuração (OWASP A05)

### a) Segredos no código-fonte
- **Procure:** chaves de API, senhas de banco, segredo de JWT, credenciais de cloud hardcoded no
  código ou commitados no repositório; `.env` versionado (não está no `.gitignore`).
- **FAIL:** qualquer segredo em texto no repositório. Repositório privado **não** é proteção —
  vaza por fork, backup, acesso indevido, histórico do git.
- **Ação:** segredo exposto no histórico do git é **considerado comprometido** — a correção não é
  só remover, é **rotacionar a credencial**. Aponte isso.
- **PASS:** segredos em variáveis de ambiente / cofre; `.env` no `.gitignore`; `.env.example`
  apenas com placeholders.
- **Ao reportar, nunca cole o valor do segredo** — só `arquivo:linha` + o tipo de credencial (ver
  exceção à regra de evidência no `SKILL.md`).

### a.1) Segredo vazando pro bundle do frontend
Vetor distinto do anterior: a variável existe só em `.env`/config do servidor, mas é **injetada no
JS enviado ao navegador** por engano — qualquer um vê via devtools/`view-source`, sem precisar de
acesso ao repositório.
- **Procure:** prefixo de env var que o framework expõe ao client por convenção (`NEXT_PUBLIC_*`
  no Next.js, `VITE_*` no Vite, `REACT_APP_*` no CRA) aplicado a um segredo real (chave de API
  privada, segredo de servidor) em vez de só a valores já públicos; `DefinePlugin`/webpack
  injetando env inteiro no bundle; chamada de API que devolve o objeto de config completo do
  servidor (incluindo chaves) para o cliente.
- **FAIL:** segredo de servidor acessível no bundle JS entregue ao navegador (inspecione o
  JS compilado/minificado, não só o código-fonte — o segredo pode não aparecer no source mas
  aparecer no build).
- **PASS:** só valores já públicos (ex.: chave pública de analytics) usam o prefixo client-side;
  segredo de servidor nunca atravessa para o bundle.

### b) CORS
- **Procure:** `Access-Control-Allow-Origin: *` combinado com credenciais, ou `origin: true`
  (reflete qualquer origem) — abre a API para qualquer site.
- **PASS:** allowlist de origens vinda de configuração/env, não wildcard quando há credenciais.

### c) Erros verbosos
- **Procure:** stack trace, query SQL, caminho de arquivo, versão de framework vazando na resposta
  de erro em produção. (No seu contexto ASP Classic, isto se liga direto ao `existingResponse`/
  página de erro customizada — erro verboso entrega mapa do sistema ao atacante.)
- **PASS:** erro genérico ao cliente em produção; detalhe só no log server-side.

### d) Superfície exposta
- Endpoints de admin/debug/monitoração (phpinfo, actuator, painéis) acessíveis publicamente;
  listagem de diretório habilitada; métodos HTTP desnecessários (TRACE, PUT) abertos.
- **PASS:** superfície mínima; painéis atrás de auth/rede interna; listagem de diretório off.

---

## 3. Cabeçalhos de segurança HTTP

Baratos, alto retorno, frequentemente ausentes. Procure e recomende:

| Header | Protege contra | Valor de referência |
|---|---|---|
| `Content-Security-Policy` | XSS, injeção de recurso | Restritiva; evitar `unsafe-inline` em script |
| `Strict-Transport-Security` | Downgrade p/ HTTP | `max-age=31536000; includeSubDomains` |
| `X-Content-Type-Options` | MIME sniffing | `nosniff` |
| `X-Frame-Options` / CSP `frame-ancestors` | Clickjacking | `DENY` ou allowlist |
| `Referrer-Policy` | Vazamento de URL | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | Abuso de APIs do browser | Desabilitar câmera/mic/geo se não usa |

Existem middlewares que setam a maioria de uma vez (Helmet no Node, equivalentes por stack), mas o
princípio é o mesmo em qualquer servidor — dá pra setar direto na config do IIS/Apache/Nginx.

---

## 4. Logging e monitoração (OWASP A09)

Falha dupla: **não logar o que importa** (cego a ataque em andamento) e **logar o que não devia**
(PII/segredo virando novo vazamento).

### O que procurar
- **Eventos de segurança logados?** Login (sucesso/falha) com IP e timestamp, acesso negado
  (403), rate limit atingido, ações administrativas, modificações de dado sensível. Ausência =
  cego a incidente (foi exatamente o gap num incidente anterior — sem log de evento, a origem do
  ataque fica invisível).
- **PII/segredo no log?** Senha, token, CPF, e-mail em texto no log = novo vetor de vazamento.
  Devem ser **mascarados**.
- **Integridade e retenção.** Logs graváveis/apagáveis pelo próprio usuário da aplicação; sem
  rotação; sem centralização.

### Critério
- **WARN/FAIL:** eventos de segurança não logados, OU PII/segredo em texto no log.
- **PASS:** eventos de segurança logados com contexto; PII mascarada; logs protegidos e rotados.

---

## 5. Componentes vulneráveis (OWASP A06)

### O que procurar
- Dependências desatualizadas com CVE conhecida (`npm audit`, `composer audit`, equivalentes).
- Framework/runtime fora de suporte (ex.: versões de PHP/Node em EOL, Zend Framework — que está
  descontinuado, o que reforça o valor da migração que você já conduz).
- `lock file` não commitado (build não reproduzível → versão vulnerável entra sem controle).

### Critério
- **FAIL:** dependência com CVE crítica/alta explorável no caminho usado; runtime em EOL exposto.
- **WARN:** desatualização sem CVE crítica confirmada, ou audit não roda no CI.
- **PASS:** deps atualizadas, audit no pipeline, lock file versionado, runtime suportado.

### Nota sobre legado sem suporte (relevante ao seu contexto)
ASP Classic e Zend descontinuado não recebem patch. Para esses, a mitigação não é "atualizar a
dependência" (não há) — é **compensar por camada**: WAF/regras no servidor, isolamento de rede,
egress restrito, e priorizar a migração dos módulos mais expostos. Trate a existência do legado em
EOL como risco documentado, não como algo "resolvível" no próprio legado.

---

## Resumo

| Classe | Falha típica | Defesa |
|---|---|---|
| Cripto senha | MD5/SHA1/plaintext | bcrypt/scrypt/Argon2 |
| PII repouso | Sem cifra | AES-GCM + chave em cofre/env |
| Trânsito | HTTP, sem HSTS | HTTPS forçado + HSTS |
| Segredos | Hardcoded/commitado | Env/cofre; rotacionar se vazou |
| CORS | Wildcard c/ credenciais | Allowlist de origem |
| Erros | Stack trace em prod | Erro genérico ao cliente |
| Headers | Ausentes | CSP/HSTS/nosniff/frame |
| Logging | Sem evento ou com PII | Logar segurança, mascarar PII |
| Deps | CVE/EOL | Audit no CI; compensar legado por camada |
