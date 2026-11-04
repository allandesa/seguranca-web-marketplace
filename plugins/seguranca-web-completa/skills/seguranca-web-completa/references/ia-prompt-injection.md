# Prompt injection e segurança de IA/LLM (diagnóstico + correção)

Se o alvo tem qualquer componente que manda texto para um LLM (chatbot, agente, RAG, MCP,
automação de e-mail/web com IA), esta classe se aplica. Ela é diferente das outras porque a falha
não é um bug localizável — é uma propriedade de como LLMs funcionam.

---

## Princípio que governa tudo (não negociável)

**Prompt injection não tem solução completa na camada de prompt.** O LLM processa instruções e
dados no mesmo fluxo de tokens, então instrução do tipo "ignore comandos maliciosos" é contornável
por reformulação adversária. Red-teams atingem taxas de bypass altíssimas contra defesa só de
prompt.

A defesa real é **arquitetural**, nesta ordem de prioridade (nunca o inverso):
1. **Arquitetura** — separação de privilégios, quebra da trifecta letal. Onde o ataque vira
   impossível ou inócuo.
2. **Limites determinísticos** — egress allowlist, caps de tokens/tool-calls, human-in-the-loop.
   Reduz o raio de explosão.
3. **Detecção** — classificador, guard LLM, delimitação de prompt. Ganha tempo, NÃO dá segurança.
   Última linha, nunca a primeira.

Se alguém pede "só liga o filtro de prompt injection", a resposta correta é: não existe esse botão
para um sistema que viola a trifecta. Conserta-se a arquitetura primeiro.

---

## DIAGNÓSTICO

### Passo 1 — Auditar a Trifecta Letal (Willison)

Liste cada ferramenta/integração do agente e marque quais das três capacidades ela toca:

- **A — Acesso a dados privados** (lê e-mail, arquivo, DB, dado de cliente, repositório).
- **B — Exposição a conteúdo não-confiável** (qualquer texto/imagem que um atacante consiga pôr no
  caminho do modelo — e-mail, PDF, página, ticket, registro de DB, alt-text de imagem).
- **C — Comunicação externa / mudança de estado** (HTTP de saída, envio de e-mail/mensagem, criar
  PR, gravar em sistema, **renderizar link ou imagem clicável** — isso já é canal de exfiltração).

**Regra:** um mesmo caminho de execução com **A + B + C** é explorável. Ponto. A meta é garantir
que pelo menos um círculo esteja ausente em qualquer caminho que toque conteúdo não-confiável.
Variante prática ("Agents Rule of Two"): um agente deve satisfazer no máximo **dois** de
{processa input não-confiável, acessa sistema sensível, muda estado externo}.

Produza o veredito explícito por caminho: "VIOLA a trifecta (A+B+C)" ou "OK — falta X". Não avance
sem isso.

### Passo 2 — As quatro classes de ameaça

| Classe | O que é | Pergunta-chave |
|---|---|---|
| **Injeção direta** | Usuário digita payload tentando sobrescrever instruções | O usuário muda o comportamento só pedindo? |
| **Injeção indireta** (a mais grave) | Payload escondido em conteúdo que o LLM ingere | De onde vem o texto que o modelo lê além do usuário? |
| **Extração** | Vazar system prompt, regras, segredos, dados de outra sessão | O que o modelo "sabe" que não deveria repetir? |
| **Abuso de escopo + consumo** | Usar a IA fora do propósito; loops/prompts caros | O que acontece com 10k requisições ou um loop recursivo? |

A **injeção indireta** é a mais relevante em produção: um único documento envenenado compromete
todo usuário que pedir para a IA processá-lo. É um confused-deputy — o agente usa os privilégios do
usuário legítimo para executar a ordem do atacante, e o usuário nunca vê. Não foque só no campo de
chat; mapeie todo conteúdo externo que o modelo lê.

### Classificação (semáforo)
- 🔴 Caminho viola a trifecta (A+B+C) sem gate.
- 🟠 Rule of Two: 2 de 3, aceitável só com gate humano/determinístico no ponto consequente.
- 🟡 Suspeito: não consegui mapear todo o conteúdo não-confiável que entra.
- 🟢 Fora da trifecta (falta A, B ou C comprovadamente).

---

## CORREÇÃO (só sob pedido explícito)

### 1. Quebrar a trifecta (a defesa que funciona)
- **Privilégio mínimo por ferramenta:** resumo de e-mail não precisa de escrita nem rede; análise
  de documento não precisa de rede. Corte a capacidade na raiz.
- **Remover C:** sem egress e sem render de conteúdo não sanitizado → sem exfiltração.
- **Remover A:** o componente que lê conteúdo sujo não tem credencial para dado sensível.
- **Segregar B:** conteúdo não-confiável é sanitizado/estruturado e nunca entra no caminho
  privilegiado como instrução livre.

### 2. Separação de processos (Dual-LLM)
- **LLM quarentenado:** vê o conteúdo sujo, extrai/estrutura, **sem ferramentas e sem privilégio**.
  Saída tratada como dado, nunca como instrução.
- **LLM privilegiado:** orquestra ações, nunca vê o texto bruto — só recebe dados tipados/validados
  do quarentenado. A instrução injetada fica presa do lado sem poder.

### 3. Taint tracking + gate
A partir do momento em que o agente ingere tokens controláveis por atacante, considere o turno
contaminado. Se contaminado, **bloqueie ou exija aprovação humana** para qualquer ação com
potencial de exfiltração. Human-in-the-loop obrigatório para ação irreversível (pagamento,
exclusão, envio externo), mostrando **a ação concreta**, não um resumo que o modelo escreveu (o
resumo pode estar envenenado).

### 4. Limites determinísticos (raio de explosão)
- **Egress allowlist:** o sistema só fala com domínios aprovados — inclusive no render de link/
  imagem (uma `<img>` com dado na query string já é exfiltração). Mata a maior parte da
  exfiltração indireta.
- **Inspeção de saída:** varrer por formatos de credencial, tokens, PII, strings de alta entropia
  antes de sair.
- **Consumo (denial-of-wallet):** limite de tamanho de input, cap de tokens de saída, cota por
  usuário/sessão, **cap de profundidade e número de tool-calls por turno**, detecção de loop,
  timeouts. Rate limit sozinho NÃO basta — não impede um único loop recursivo de queimar recurso
  num burst. Trate limite de token como controle de segurança, não linha de orçamento.

### 5. Detecção e prompt (última camada, com ressalva)
Só depois do acima. Rotule sempre como **redução de ataque oportunista, não segurança**: nunca
ponha segredo no system prompt (assuma que vaza); delimite conteúdo não-confiável explicitamente;
guard LLM/classificador na frente; recuse revelar system prompt. Isolamento por tenant é
arquitetural, não de prompt.

---

## Antipadrões
- Confiar em instrução de prompt para conter ação consequente.
- Tratar multiagente como mais seguro — agentes compartilham/vazam contexto; para a auditoria,
  trate o conjunto como um só agente.
- "O filtro do fornecedor cobre" — cobre padrão conhecido; payload parafraseado passa.
- Renderizar markdown/imagem/link de conteúdo não-confiável sem ver que é canal de exfiltração.
- Validar a feature de IA "por vibe", sem red-team com ataque adaptativo.

## Calibração
- **Fato/consenso:** injeção é arquitetural; trifecta; bypass de filtro por reformulação. Afirme.
- **Boa prática:** privilégio mínimo, egress allowlist, human-in-the-loop, Dual-LLM.
- **Promissor, não garantido:** guard LLMs, instruction hierarchy, CaMeL. Apresente como redução
  de probabilidade.
- **Nunca afirme** "seguro contra injeção". Declare o raio de explosão residual.
