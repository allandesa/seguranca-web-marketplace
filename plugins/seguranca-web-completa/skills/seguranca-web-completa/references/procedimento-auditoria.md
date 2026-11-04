# Procedimento de auditoria — read-only, rodadas e inventário exaustivo

Este arquivo define COMO auditar com rigor, independente da classe de falha. É o que separa uma
auditoria confiável ("colei o código que prova cada achado") de um palpite ("parece inseguro").

---

## 1. Contrato READ-ONLY (modo diagnóstico — o padrão)

Enquanto estiver auditando, a skill opera sob trava absoluta. Regras que não se negociam:

1. **Não editar, criar, mover ou apagar nenhum arquivo.** Zero alteração no código do alvo.
2. **Não rodar o servidor nem fazer chamadas externas** contra o alvo (a menos que seja um
   teste ativo explicitamente autorizado — ver `pentest-plano.md` e a fronteira ética).
3. **Não escrever correção.** Achou falha, descreve a falha — não escreve o fix. A correção é uma
   fase separada, que só acontece quando o usuário pedir explicitamente.
4. **Não afirmar que algo está protegido sem colar o código que protege.** Sem evidência
   rastreada (`arquivo:linha` + trecho) = **NÃO VERIFICADO**, nunca "OK".

Por que a trava importa: o principal modo de falha de uma auditoria feita por LLM é "consertar de
passagem" — modificar o que devia só ler, misturando diagnóstico com mudança não revisada. A
trava elimina isso. O segundo modo de falha é afirmar segurança por otimismo; a regra 4 elimina
esse.

Ao concluir uma rodada de diagnóstico: **pare e aguarde revisão.** Não emende com correções.

---

## 2. Auditoria em rodadas (decida no início)

Auditar tudo de uma vez, num alvo grande, produz varredura rasa — cada classe recebe atenção
superficial. Fatiar por classe produz profundidade. **No início de cada auditoria, pergunte ao
usuário** qual modo ele quer:

- **Rodada única:** cobre todas as classes de uma vez. Bom para alvo pequeno ou visão geral.
- **Por rodadas:** uma classe por vez, cada uma com relatório próprio, parando para revisão entre
  elas. Bom para alvo grande, ou quando ele quer priorizar (ex.: "rodada 1: auth/authz/IDOR").

Rodadas típicas, na ordem de raio de dano (auth primeiro porque quebra tudo o resto):
1. Autenticação, autorização e IDOR
2. Injeção (SQL/NoSQL/comando/XSS/SSRF)
3. Upload, path traversal e permissões de arquivo
4. Criptografia, segredos e configuração
5. Logging, dependências e conformidade

Cada rodada usa a referência de diagnóstico correspondente. Ao fim de cada uma, relatório +
"aguardando revisão", sem avançar sozinho para a próxima.

---

## 3. Método de rigor — três passos por rodada

Aplique dentro de qualquer rodada. É o que sua auditoria de auth/authz/IDOR trouxe, generalizado.

### Passo A — Documentar o padrão pretendido (baseline)
Antes de auditar item por item, localize e leia o mecanismo que *deveria* proteger: o middleware
de auth, o de autorização/ownership, o de validação, o de escape — onde e como são registrados
(global? por router? por rota?). Documente em uma frase: **"O padrão pretendido é X."** Isso vira
o baseline. Auditar contra um baseline detecta **desvio** ("esta rota não segue o padrão"), que é
mais preciso e mais rápido que julgar cada item isolado do zero.

### Passo B — Inventário exaustivo
Varra **todos** os pontos relevantes da classe e produza uma lista completa — não uma amostra.
Para auth/rotas, a tabela é:

| Método | Path | Arquivo:linha | Handler |

"Não pule nenhuma." Se houver itens montados dinamicamente (rotas por spread, handlers gerados,
queries construídas em runtime), **sinalize** — o que não dá para inventariar estaticamente é
risco por si só, porque escapa da auditoria.

O inventário exaustivo é o que impede o "achado por vibe": você não classifica o que não listou, e
lista tudo.

### Passo C — Rastrear a proteção real e colar a evidência
Para cada item do inventário, rastreie a cadeia que **efetivamente roda** antes do ponto sensível
(o handler, a query, o render). **Cole o trecho real** que aplica — ou não aplica — a proteção.
Classifique com evidência; sem trecho colado, o veredito é NÃO VERIFICADO.

---

## 4. Classificação (semáforo)

Use este esquema; ele mapeia direto para o status do relatório (FAIL/WARN/PASS/N-V).

- 🔴 **SEM PROTEÇÃO** — aceita a operação sensível sem o controle que deveria ter (rota sem auth,
  query concatenada, upload executável). = FAIL.
- 🟠 **PARCIAL** — tem uma camada mas falta outra (autenticada mas sem checar direito àquele
  recurso; validação parcial; escape num contexto mas não noutro). = FAIL ou WARN conforme
  explorabilidade.
- 🟡 **SUSPEITO** — indício de falha (recebe id e usa no banco; padrão inseguro presente) mas não
  confirmei a exploração end-to-end. = WARN.
- 🟢 **PROTEGIDA** — controle presente e comprovado, **com o trecho colado**. = PASS.
- ⚪ **NÃO VERIFICADO** — não consegui rastrear com certeza. Nunca vira 🟢 por otimismo.

**Confiança por achado:** CONFIRMADO (rastreei a cadeia inteira e não há checagem) | SUSPEITO
(indício, não confirmei 100%).

---

## 5. Priorização por raio de dano

Ordene 🔴🟠🟡 por impacto real, não por gravidade teórica. Priorize explicitamente:
- **Endpoints que recebem payload externo/automação** (webhooks, integrações tipo n8n): pergunta
  central — *quem consegue chamar? só a credencial autorizada ou qualquer um?* Endpoint de
  automação "aberto porque é interno" é achado de alta prioridade; "interno" evapora no primeiro
  proxy mal configurado.
- **Endpoints que leem/escrevem dado pessoal** (exposição = risco LGPD, incidente reportável).
- **Upload** (vetor de RCE e de malware servido a terceiros).
- **Os próprios endpoints de auth** (login, reset, emissão de token) — comprometê-los compromete
  tudo.

Um id sequencial exposto num endpoint de dado pessoal sem checagem de posse é mais urgente que uma
falha de header num endpoint público sem dado — mesmo que ambos sejam "achados". Contextualize
pelo alcance do atacante e pela sensibilidade do dado.
