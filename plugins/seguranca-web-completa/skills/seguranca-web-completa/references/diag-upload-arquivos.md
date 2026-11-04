# Upload de arquivos, path traversal e permissões de pasta

Duas classes que quase sempre andam juntas e que você citou explicitamente: **upload inseguro** e
**permissões de pasta/arquivo**. A ponte entre elas é o **path traversal** — o mecanismo pelo qual
input do usuário vira caminho no sistema de arquivos.

---

## 1. Upload inseguro de arquivos

O upload é uma das superfícies mais perigosas porque, se mal feito, transforma "receber um
arquivo" em "executar código do atacante no servidor" ou "servir malware para outros usuários".

### Os vetores (procure cada um)

**a) Execução do arquivo enviado (o pior caso).**
Se o arquivo enviado cai num diretório servido pelo servidor web E pode ser interpretado (`.php`,
`.asp`, `.aspx`, `.jsp`, `.phtml`, `.cshtml`), o atacante sobe um webshell e executa código.
- **Procure:** diretório de upload dentro da webroot (`/public`, `wwwroot`, servido por
  IIS/Apache/Nginx) sem bloqueio de execução.
- **FAIL crítico** se confirmado.

**b) Validação de tipo por blocklist ou por extensão/Content-Type do cliente.**
Toda blocklist de extensão tem furo (`.pHp`, `.php5`, `.phtml`, dupla extensão `foto.jpg.php`,
null byte em stacks antigas). Content-Type vem do cliente e é forjável. Nome do arquivo é
controlado pelo atacante.
- **Defesa correta:** **allowlist** de extensões + verificação do conteúdo real (magic
  bytes/MIME server-side) + renomear o arquivo para nome gerado pelo servidor.

**c) Diretório de upload sem isolamento.**
Mesmo arquivos "inofensivos" (imagens) podem carregar payload (SVG com script → XSS; polyglot).
- **Defesa:** servir uploads de domínio/subdomínio separado sem cookies de sessão, ou via
  endpoint que força `Content-Disposition: attachment` e `Content-Type` correto; nunca deixar o
  servidor adivinhar o tipo.

**d) Sem limite de tamanho/quantidade.**
Vira DoS (encher disco) e denial-of-wallet em storage cobrado.
- **Defesa:** limite de tamanho por arquivo e cota por usuário/sessão.

### Critério de classificação
- **FAIL:** upload cai em diretório executável, OU validação é só por extensão/Content-Type do
  cliente, OU nome do arquivo do cliente é usado direto no path.
- **WARN:** allowlist de extensão presente mas sem verificação de conteúdo real; ou sem limite de
  tamanho; ou sem isolamento de domínio.
- **PASS:** allowlist + verificação de magic bytes + rename server-side + diretório fora da
  webroot ou com execução desativada + limite de tamanho.

### Checklist de upload seguro (o alvo da correção)
1. Allowlist de tipos, validada pelo **conteúdo** (magic bytes), não só pela extensão.
2. Nome do arquivo **gerado pelo servidor** (UUID), extensão derivada do tipo validado.
3. Diretório de destino **fora da webroot** ou com execução de script **desativada** na config
   do servidor.
4. Limite de tamanho e cota.
5. Servir via domínio isolado ou forçando download; nunca inline com tipo adivinhado.
6. Antivírus/scan quando o arquivo é compartilhado com terceiros.

---

## 2. Path Traversal (Directory Traversal)

Input do usuário compõe um caminho de arquivo, e o atacante usa `../` (ou variações codificadas:
`..%2f`, `..%252f`, `....//`, encoding unicode) para escapar do diretório pretendido e ler/
escrever arquivos arbitrários (`/etc/passwd`, `web.config`, `.env`, arquivos de outro usuário).

### O que procurar
Qualquer ponto onde input vira parte de um path:
- Download/leitura de arquivo por nome/id vindo do usuário: `readFile(baseDir + req.query.file)`,
  `Server.MapPath(request("arquivo"))` (ASP), `include $_GET['page']`.
- Include/require dinâmico (**LFI — Local File Inclusion**): `include($_GET['modulo'].".php")`.
  Em PHP clássico isso é vetor de RCE clássico.
- **RFI (Remote File Inclusion):** include de URL controlada por input — raro hoje mas devastador.

### Critério
- **FAIL crítico:** input do usuário concatenado em path de leitura/escrita/include sem
  normalização e sem confinamento ao diretório base.
- **WARN:** há sanitização de `../` mas por replace simples (contornável por dupla codificação) em
  vez de normalização canônica + verificação de que o path resolvido está dentro do base.
- **PASS:** o caminho final é **canonicalizado** e verificado como estando dentro do diretório
  permitido; ou o input só seleciona de uma **allowlist** de arquivos/ids, nunca compõe o path
  livremente.

### Defesa (o padrão correto)
Não confie em remover `../`. **Resolva o caminho absoluto (canonical) e verifique que ele começa
com o diretório base permitido.** Melhor ainda: mapeie o input do usuário para um id/chave e
resolva o path a partir de uma allowlist interna — o usuário nunca fornece o path, só escolhe de
uma lista.

---

## 3. Permissões de pasta e arquivo (sistema de arquivos)

Falha de configuração no nível do SO/servidor, não do código. Frequentemente ignorada e explorável.

### O que procurar

**a) Diretórios sensíveis dentro da webroot e acessíveis via HTTP.**
`.env`, `.git/`, `config.php`, `web.config` com segredos, backups (`.bak`, `.sql`, `.zip`),
`composer.json`/`package.json`, arquivos de log. Se estão sob a webroot, podem ser baixados
direto pela URL.
- **Teste:** tentar `https://alvo/.env`, `/.git/config`, `/backup.sql`. **FAIL** se retornar
  conteúdo.

**b) Permissões excessivas no filesystem.**
- `777` (rwx para todos) em diretórios de aplicação — qualquer processo no host escreve/executa.
- Diretório de upload com permissão de **execução**.
- Arquivos de config/segredo legíveis por usuários/processos que não deveriam (`.env` com `644`
  num host compartilhado).
- **Defesa:** privilégio mínimo. Aplicação roda como usuário dedicado sem privilégio; arquivos de
  config `600`/`640` restritos ao dono; diretórios de upload sem execução; nada sensível legível
  pelo mundo.

**c) Ownership e processo.**
Servidor web rodando como root/administrador é escalada garantida se houver qualquer RCE.
- **Defesa:** processo do servidor roda com conta de serviço de baixo privilégio.

**d) Config específica de servidor (contexto Windows/IIS — relevante para ASP Classic).**
- Diretório de upload com "Script/Execute" habilitado no IIS = executa `.asp` enviado. Deve ser
  só "Read".
- Application Pool Identity com privilégio além do necessário.
- `web.config` legível ou com detalhes de erro expostos.

### Critério
- **FAIL:** arquivo sensível baixável via HTTP; upload em diretório executável; servidor rodando
  como root/admin; permissão `777` em código.
- **WARN:** permissões mais frouxas que o necessário mas sem exposição direta confirmada.
- **PASS:** sensíveis fora da webroot ou bloqueados; privilégio mínimo no processo e nos arquivos;
  upload sem execução.

### Defesa (aponte para hardening + config de servidor)
Privilégio mínimo em processo e arquivos; sensíveis fora da webroot ou bloqueados por regra do
servidor; upload sem execução; conta de serviço dedicada de baixo privilégio.

---

## Resumo

| Falha | Mecanismo | Defesa primária |
|---|---|---|
| Upload → RCE | Arquivo executável em dir servido | Allowlist + magic bytes + rename + dir sem execução |
| Upload → XSS/malware | Tipo adivinhado, SVG/polyglot | Domínio isolado / forçar download / MIME correto |
| Path traversal / LFI | `../` em path/include | Canonicalizar + confinar ao base; ou allowlist de ids |
| Arquivo sensível exposto | `.env`/`.git`/backup sob webroot | Fora da webroot ou bloqueio no servidor |
| Permissão excessiva | `777`, execução em upload, root | Privilégio mínimo em processo e filesystem |
