# Template do relatório de auditoria de segurança

Use esta estrutura como entregável do Passo 3. Cada achado é rastreável (aponta onde está) e
honesto (declara confiança e cobertura). Nada de "está seguro" — declare o risco residual.

---

## 1. Escopo e cobertura (declaração honesta — vem primeiro)

- **Alvo:** [aplicação/API, ambiente]
- **Stack identificada:** [backend, dados, auth, servidor]
- **O que foi auditado:** [rotas/módulos/classes cobertas]
- **O que NÃO foi coberto:** [partes não inspecionadas e por quê]
- **Método:** [estático / ativo / ambos; com ou sem acesso ao código]

> Frase obrigatória no lugar de "sistema seguro": "Não foram encontradas falhas nas classes
> [X, Y, Z] dentro do escopo auditado. As classes/módulos [W] não foram cobertos e permanecem sem
> veredito."

## Parada obrigatória

Ao fim da rodada, **pare e aguarde revisão**. Não escreva correções no relatório de diagnóstico —
a correção é uma fase separada, iniciada só quando o usuário pedir explicitamente.

## 2. Inventário exaustivo (todos os itens da classe, sem amostragem)

Antes dos achados, a lista completa. Para rotas:

| Método | Path | Arquivo:linha | Handler | Classificação |
|---|---|---|---|---|
| GET | /api/candidatos/:id | rotas.js:42 | listarCandidato | 🟡 SUSPEITO |

Sinalize itens montados dinamicamente (não inventariáveis estaticamente) — são risco por si só.

## 3. Tabela-resumo

| # | Achado | Classe OWASP | Semáforo | Severidade | Confiança |
|---|---|---|---|---|---|
| 1 | ex.: SQLi em /busca | A03 Injeção | 🔴 | Crítica | CONFIRMADO |
| 2 | ex.: upload sem verificação de conteúdo | A04/A05 | 🟠 | Alta | SUSPEITO |

**Semáforo:** 🔴 sem proteção / 🟠 parcial / 🟡 suspeito / 🟢 protegida (com evidência) / ⚪ não
verificado. **Confiança:** CONFIRMADO / SUSPEITO.

Contagem por categoria: 🔴 X | 🟠 X | 🟡 X | 🟢 X | ⚪ X.

## 3. Achados detalhados (por severidade, crítico primeiro)

Para cada FAIL e WARN:

### [#] Título do achado — [Severidade] — [Confiança]
- **Onde:** arquivo/rota/config exata (a evidência). Se não localizado, o motivo do WARN.
- **O que está errado:** a falha em uma frase técnica.
- **Cenário de ataque:** como um atacante explora, e o que ganha (o impacto concreto).
- **Alcance:** público / autenticado / requer privilégio — o que contextualiza a severidade.
- **Correção:** o padrão que resolve + **ponteiro para a skill/seção** que implementa
  (`hardening-seguranca-web` / `defesa-prompt-injection` / config de servidor).

## 4. Score

- Total de checks: X
- PASS: X | FAIL: X | WARN: X | N/A: X
- Cobertura: X% (das classes aplicáveis ao alvo)

## 5. Ordem de correção priorizada

Lista numerada por severidade e facilidade de exploração. Racional: corrige-se primeiro o que é
**crítico E alcançável por atacante não autenticado**, depois o resto. Cada item aponta a skill/
seção de correção.

1. [Crítico, público] ...
2. [Crítico, autenticado] ...
3. [Alto] ...

## 6. Risco residual (o que sobra após as correções propostas)

Declaração honesta: o que *não* dá para eliminar no escopo/prazo (ex.: legado em EOL sem patch),
e qual controle compensatório contém o raio de explosão. Isto substitui qualquer promessa de
"100% seguro".

## 7. Conformidade (quando aplicável — contexto BR/SaaS)

Se processa dado pessoal: amarrar achados de exposição/vazamento à **LGPD** (exposição de PII por
falha de segurança é incidente reportável). Para SaaS/auditoria externa, mapear para SOC 2 / ISO
27001 conforme o requisito do cliente.
