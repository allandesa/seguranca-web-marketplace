---
name: seguranca-web-completa
description: >
  Auditoria de segurança completa e autossuficiente de aplicação web/API, agnóstica de stack
  (PHP/ASP Classic/Zend/Laravel, Node/Bun, Python, .NET). Cobre todas as classes OWASP: controle
  de acesso e IDOR, injeção SQL/NoSQL/comando/LDAP, XSS, SSRF, upload inseguro, path traversal e
  permissões de pasta/arquivo, criptografia, segredos, config e headers, auth e sessão, logging,
  dependências, prompt injection em IA/LLM/agente, e LGPD. Opera READ-ONLY por padrão (só
  diagnostica, com evidência rastreada por arquivo:linha), suporta auditoria em rodadas, faz
  inventário exaustivo de rotas, e só escreve correções sob pedido explícito. Use SEMPRE que
  pedir: auditoria ou revisão de segurança, pentest, análise de vulnerabilidades, "é seguro?",
  checar OWASP, revisar rota/endpoint/upload, IDOR, SQL injection, XSS, permissões de arquivo,
  segurança de feature com IA, ou avaliar risco antes de produção. Use também proativamente ao
  descrever feature que recebe input, faz upload, monta query, expõe rota ou usa LLM.
---

# Segurança Web — Auditoria Completa

## O que esta skill é

Uma skill **autossuficiente** de segurança. Ela diagnostica todas as classes de falha de uma
aplicação web/API e, quando pedido, corrige. Não depende de nenhuma outra skill.

Dois modos, e o diagnóstico é sempre o padrão:

- **Modo AUDITORIA (padrão, READ-ONLY):** mapeia e classifica falhas com evidência rastreada.
  Não toca em arquivo, não roda o servidor, não escreve correção.
- **Modo CORREÇÃO (só sob pedido explícito):** entra apenas quando o usuário disser claramente
  "corrija", "implemente o fix", "aplique o hardening". Enquanto ele não pedir, a skill permanece
  em diagnóstico — achou falha, descreve a falha, não escreve o conserto.

## Princípios não negociáveis

1. **Evidência ou nada.** Todo achado aponta `arquivo:linha` + o trecho que prova. Toda afirmação
   de "protegido" cola o código que protege. Sem evidência rastreada = **NÃO VERIFICADO**, nunca
   "OK". Isso vale nos dois sentidos: não afirme falha sem localizar, nem segurança sem colar a
   defesa.
   **Exceção:** para segredo/credencial exposta, a evidência é `arquivo:linha` + o **tipo** da
   credencial (ex.: "chave de API da Stripe", "senha de banco") — **nunca o valor em si**. Colar o
   valor no relatório cria um segundo vazamento (o próprio relatório passa a guardar o segredo).

2. **Ausência de prova não é prova de segurança.** Não achar a falha pode ser ausência dela — ou
   que você não olhou no lugar certo. Declare a cobertura real. Nenhuma auditoria de escopo
   parcial autoriza dizer "o sistema está seguro".

3. **Severidade = probabilidade × impacto × alcance.** A mesma falha é crítica numa rota pública
   e média atrás de auth de admin. Contextualize pelo alcance do atacante e pela sensibilidade do
   dado, nunca pela gravidade teórica isolada.

4. **Diagnóstico e correção não se misturam.** Auditar e consertar na mesma passada é o dev
   revisando o próprio PR — o viés mascara a falha. Diagnostica com evidência; corrige depois, se
   e quando pedido.

## Fluxo de trabalho

### Passo 0 — Definir o modo e o escopo (pergunte no início)

No começo de cada auditoria, estabeleça:
- **Rodada única ou por rodadas?** Alvo grande audita-se por classe, uma de cada vez, com
  relatório e revisão entre elas — fatiar evita varredura rasa. Alvo pequeno cabe numa rodada.
  Ver `references/procedimento-auditoria.md` (seção rodadas).
- **Escopo:** qual aplicação/módulo, o que está incluído e excluído.

Confirme a trava READ-ONLY explicitamente ao usuário antes de começar.

### Passo 1 — Reconhecimento e baseline

Mapeie a stack (linguagem/framework, camada de dados, mecanismo de auth, servidor web) e a
superfície (rotas, o que recebe input, o que faz upload, o que toca o filesystem, o que fala com
serviços externos, onde há IA/LLM). **Mapeie também os dados sensíveis:** quais existem (PII,
senha, dado financeiro/saúde, segredo), por onde circulam, onde ficam armazenados e quem/o que
pode acessá-los — esse mapa é o que alimenta a priorização por raio de dano (
`references/procedimento-auditoria.md`) e a seção de LGPD. Depois, para a classe da rodada,
**documente o padrão pretendido** em uma frase ("O padrão de auth é X") — o baseline contra o qual
você detecta desvio. Método detalhado em `references/procedimento-auditoria.md`.

### Passo 2 — Inventário exaustivo

Liste **todos** os pontos da classe, sem amostragem. Para rotas: tabela `Método | Path |
Arquivo:linha | Handler`, não pule nenhuma. Sinalize o que é montado dinamicamente (rotas por
spread, queries construídas em runtime) — o que escapa do inventário estático é risco por si só.

### Passo 3 — Auditar por classe (rastrear proteção real + colar evidência)

Para cada item do inventário, rastreie a cadeia que efetivamente roda antes do ponto sensível e
cole o trecho real. Classifique com o semáforo (🔴 sem proteção / 🟠 parcial / 🟡 suspeito /
🟢 protegida-com-evidência / ⚪ não verificado) e a confiança (CONFIRMADO / SUSPEITO). Consulte a
referência da classe:

| Classe | Referência de diagnóstico |
|---|---|
| Auth, autorização, IDOR, RBAC, sessão, JWT | `references/diag-controle-acesso.md` |
| Injeção SQL/NoSQL/comando/LDAP, XSS, SSRF, prototype pollution | `references/diag-injecao.md` |
| Upload, path traversal, permissões de pasta/arquivo, LFI/RFI | `references/diag-upload-arquivos.md` |
| Cripto, segredos, config, headers, CORS, logging, deps | `references/diag-cripto-config.md` |
| Prompt injection / IA / LLM / agente | `references/ia-prompt-injection.md` |

### Passo 4 — Teste ativo (opcional, só com autorização)

Se o usuário quer confirmar explorabilidade (sair de 🟡 SUSPEITO para 🔴 CONFIRMADO), use
`references/pentest-plano.md`. Atenção à fronteira ética lá descrita: só alvo próprio ou com
autorização por escrito, e teste ativo **quebra a trava read-only** — exige "sim" explícito do
usuário, porque faz chamadas contra o alvo.

### Passo 5 — Relatório e parada

Produza o relatório no formato de `references/relatorio-template.md`: inventário completo,
achados priorizados por raio de dano com cenário de exploração concreto, cobertura honesta e risco
residual. **Ao terminar a rodada, pare e aguarde revisão.** Não emende com correções.

### Passo 6 — Correção (SÓ quando o usuário pedir explicitamente)

Quando — e apenas quando — o usuário pedir o conserto, saia do read-only e entre em correção.
Use as referências `fix-*`, aplicando o padrão adaptado à stack + o critério de aceitação, sem
quebrar o que funciona, e reexecutando o teste relevante para confirmar que o furo fechou.

| Classe a corrigir | Referência de correção |
|---|---|
| SQLi, XSS, comando, SSRF, prototype pollution | `references/fix-injecao.md` |
| Upload, path traversal, permissões de pasta | `references/fix-upload-arquivos.md` |
| Auth, IDOR, RBAC, sessão, JWT, rate limit | `references/fix-acesso-auth.md` |
| Cripto, segredos, headers, CORS, logging, deps, LGPD | `references/fix-cripto-config.md` |
| Prompt injection / IA | `references/ia-prompt-injection.md` (seção correção) |

## Calibração de certeza (rotule cada achado)

- **CONFIRMADO:** rastreei a cadeia inteira, colei a evidência, o caminho de exploração é claro.
- **SUSPEITO:** o padrão inseguro está presente mas não confirmei a exploração end-to-end. WARN.
- **NÃO VERIFICADO:** não tive acesso/certeza. Declare o gap; nunca chute 🟢.
- **Nunca** escreva "o sistema está seguro". Escreva: "não encontrei falha nas classes X, Y, Z no
  escopo auditado; as classes/módulos W não foram cobertos".

## Antipadrões a apontar sempre

- Validação só no frontend como controle de acesso — o backend é a fronteira real.
- Tratar ORM como imunidade a SQLi (raw/query builder mal usado injeta igual).
- Blocklist de extensão/Content-Type em upload (a defesa é allowlist + conteúdo real).
- "Usamos UUID, então não tem IDOR" — UUID reduz enumeração, não é a defesa; a defesa é checar
  posse no servidor.
- "Está atrás de login, é seguro" — IDOR e escalação vivem exatamente aí.
- Endpoint de automação (webhook/n8n) "aberto porque é interno" — "interno" some no primeiro proxy
  mal configurado; ele precisa de credencial de verdade.
- Segredo no código "porque o repo é privado" — repo privado vaza; segredo que esteve no git está
  comprometido e precisa rotação.
- SHA-256+salt para senha (a defesa é bcrypt/scrypt/Argon2).
- Implementar hash, cifra, token ou verificação de assinatura **própria** em vez de usar biblioteca
  consolidada ("rolar a própria cripto/auth") — mesmo bem-intencionado, costuma ter furo que a lib
  testada não tem.
- Afirmar segurança sem colar o trecho que protege.

## Arquivos de referência

Procedimento (leia sempre no início):
- `references/procedimento-auditoria.md` — contrato read-only, rodadas, baseline, inventário
  exaustivo, semáforo de classificação, priorização por raio de dano.

Diagnóstico por classe:
- `references/diag-controle-acesso.md` — auth, IDOR, RBAC, escalação, sessão, JWT, trust no front.
- `references/diag-injecao.md` — SQL/NoSQL/comando/LDAP/SSTI, XSS, SSRF, prototype pollution.
- `references/diag-upload-arquivos.md` — upload, path traversal, permissões, LFI/RFI.
- `references/diag-cripto-config.md` — cripto, segredos, CORS, headers, logging, deps.
- `references/ia-prompt-injection.md` — prompt injection direto/indireto, extração, trifecta,
  consumo/denial-of-wallet (diagnóstico e correção).

Teste e relatório:
- `references/pentest-plano.md` — teste ativo por classe, ferramentas, fronteira ética.
- `references/relatorio-template.md` — template do relatório de achados.

Correção (só no modo correção):
- `references/fix-injecao.md`, `references/fix-upload-arquivos.md`,
  `references/fix-acesso-auth.md`, `references/fix-cripto-config.md`.
