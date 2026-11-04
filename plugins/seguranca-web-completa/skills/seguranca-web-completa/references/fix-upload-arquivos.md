# Correção — upload seguro, path traversal e permissões de pasta

As três falhas do sistema de arquivos, com o padrão de correção e o critério de aceitação.
Ilustrativo e cross-stack.

---

## 1. Upload seguro — os 6 controles (aplicar todos)

Um único controle não basta; a segurança do upload é a soma. Ordem de importância:

**1. Diretório de destino fora da webroot OU sem execução de script.**
A defesa que mais importa: mesmo que passe um `.php`/`.asp`, ele não executa.
- Guarde uploads fora do diretório servido; sirva por endpoint controlado.
- Se tiver que ficar na webroot, **desative execução** na config do servidor (IIS: remover
  Script/Execute do diretório; Apache: `php_admin_flag engine off` / remover handler; Nginx: não
  passar o diretório ao interpretador).

**2. Allowlist de tipo validada pelo CONTEÚDO real (magic bytes), não pela extensão/Content-Type.**
```
// pseudo: ler os primeiros bytes e conferir a assinatura real do arquivo
const tipoReal = detectarPorMagicBytes(buffer);        // ex.: 'image/png'
if (!tiposPermitidos.has(tipoReal)) reject();          // allowlist, nunca blocklist
```
Extensão e Content-Type vêm do cliente e são forjáveis. A verdade é o conteúdo.

**3. Nome do arquivo gerado pelo servidor (UUID), extensão derivada do tipo validado.**
Nunca use o nome fornecido pelo cliente no path — é o vetor de path traversal e de dupla extensão.

**4. Limite de tamanho e cota por usuário/sessão.**
Evita DoS de disco e denial-of-wallet em storage cobrado.

**5. Servir sem adivinhar o tipo.**
Ao entregar o arquivo, force `Content-Type` correto e, para conteúdo não-renderizável inline,
`Content-Disposition: attachment`. Melhor ainda: sirva uploads de **domínio/subdomínio separado
sem cookies de sessão** — assim um SVG malicioso não roda no contexto da aplicação.

**6. Scan (quando compartilhado com terceiros).**
Antivírus/scan de conteúdo para arquivos que outros usuários vão baixar.

**Critério de aceitação:** upload de arquivo executável não executa; validação por conteúdo real;
nome server-side; limite ativo; servido com tipo forçado. Confirme com o teste de upload da
auditoria.

---

## 2. Path traversal → canonicalizar + confinar (ou allowlist de ids)

**Padrão A (preferido): o usuário nunca compõe o path.** Ele fornece um id/chave; o servidor
resolve o caminho a partir de uma allowlist/registro interno.
```
const arquivos = { "manual": "/dados/docs/manual.pdf" };   // registro interno
const caminho = arquivos[idRecebido];                       // input só seleciona
if (!caminho) reject();
```

**Padrão B (quando o path precisa ser composto): canonicalizar e verificar confinamento.**
Não confie em remover `../` (contornável por dupla codificação). Resolva o caminho absoluto e
confirme que ele começa dentro do diretório base permitido.
```
const base = "/dados/uploads";
const resolvido = pathResolveAbsoluto(base, inputUsuario);   // canonicaliza
if (!resolvido.startsWith(base + separador)) reject();       // confina
```

**Include/require dinâmico (LFI):** elimine include de path controlado por input. Se precisa de
"módulo dinâmico", use allowlist de módulos permitidos — nunca `include(input + ".php")`.

**Critério:** input não escapa do diretório base (testado com `../` e variações codificadas); ou
input só seleciona de allowlist. Sem include de path controlado por usuário.

---

## 3. Permissões de pasta e arquivo → privilégio mínimo

Correção de postura no SO/servidor, não no código. Aplique:

**a) Sensíveis fora da webroot ou bloqueados.**
`.env`, `.git/`, config com segredo, backups (`.bak/.sql/.zip`), logs — não podem ser baixáveis
via HTTP.
- Mova para fora da webroot, **ou** bloqueie por regra do servidor:
  - IIS: `<requestFiltering>` negando as extensões/paths, ou colocar fora do site.
  - Apache: `<FilesMatch>` / `Require all denied` para os padrões.
  - Nginx: `location ~ /\.(env|git) { deny all; }`.
- Desative **listagem de diretório**.

**b) Permissões de arquivo por privilégio mínimo.**
- Código e config: legível só pelo dono/conta de serviço (`640`/`600` para segredo; nunca `777`).
- Diretório de upload: **sem** permissão de execução.
- Nada sensível legível pelo "mundo", especialmente em host compartilhado.

**c) Processo com conta de baixo privilégio.**
Servidor web **não** roda como root/Administrator. Conta de serviço dedicada, só com o acesso
necessário. No IIS, Application Pool Identity restrita.

**d) Ownership coerente.**
Arquivos da aplicação pertencem à conta de serviço; o processo não pode sobrescrever o próprio
código (reduz impacto de webshell).

**Critério:** nenhum arquivo sensível baixável via HTTP; listagem de diretório off; permissões no
mínimo necessário; upload sem execução; processo sem privilégio administrativo. Confirme tentando
baixar `/.env`, `/.git/config`, `/backup.sql` (devem falhar).

---

## Resumo

| Falha | Correção-raiz | Critério |
|---|---|---|
| Upload → RCE | Dir sem execução + allowlist por conteúdo + rename | Executável enviado não roda |
| Upload → XSS/malware | Domínio isolado / forçar download / tipo correto | SVG/polyglot não executa no contexto |
| Path traversal/LFI | Allowlist de ids, ou canonicalizar+confinar | `../` não escapa do base |
| Sensível exposto | Fora da webroot ou bloqueio no servidor | `/.env` não baixa |
| Permissão excessiva | Privilégio mínimo em processo e filesystem | Sem 777; upload sem exec; sem root |
