---
tags:
  - devops
  - git
area: devops
tipo: conteudo
prerequisites: []
next:
  - "[[11-Exercicios/git]]"
trilha: "[[00-Trilha/devops]]"
---

# Git

## O que é e por que existe

Git é um sistema de controle de versão **distribuído** criado por Linus Torvalds em 2005 para gerenciar o código do kernel Linux. Antes do Git existia o BitKeeper, que era proprietário — quando a licença foi revogada, Torvalds escreveu o Git em duas semanas com objetivos claros: velocidade, simplicidade de design, suporte a desenvolvimento não-linear (milhares de branches paralelos) e totalmente distribuído.

A palavra-chave é **distribuído**: cada clone é um repositório completo com todo o histórico — não existe dependência de um servidor central para a maioria das operações. O modelo mental correto não é "servidor com backups locais". É uma rede de repositórios pares onde qualquer um pode ser fonte da verdade. O GitHub/GitLab é uma convenção social, não uma exigência técnica.

> [!info] Por que isso importa em DevOps
> Pipelines CI/CD rodam operações Git em cada push. Entender o modelo interno evita bugs silenciosos: por que o `git fetch` não atualiza o working tree? Por que `git pull --rebase` diverge do merge? Por que o push foi rejeitado se eu tinha as mudanças? Esses detalhes quebram pipelines e causam incidentes.

---

## Como funciona internamente

### O DAG — Directed Acyclic Graph

O histórico do Git é um **grafo dirigido acíclico (DAG)**. Cada commit aponta para seu(s) pai(s) — nunca cria ciclos. Isso tem implicações diretas:

```
A ← B ← C ← D  (main)
         ↑
         E ← F  (feature/auth)
```

- O branch `main` aponta para `D`
- O branch `feature/auth` aponta para `F`
- `C` é o **merge base** — ancestral comum dos dois branches
- Um **merge** cria um novo commit com dois pais: `G` com pais `D` e `F`
- Um **rebase** pega `E` e `F` e os "recria" sobre `D`, gerando novos SHAs `E'` e `F'`

**Por que isso importa:** branches são baratos porque são apenas ponteiros para nós do grafo. Criar um branch não copia nada — é literalmente um arquivo com 40 caracteres (SHA).

### O Object Store

Git armazena tudo em `.git/objects/` como objetos **imutáveis** identificados por SHA-1 (migração para SHA-256 em andamento). Existem 4 tipos:

| Tipo | O que armazena |
|---|---|
| **blob** | conteúdo de um arquivo (sem nome, sem metadado) |
| **tree** | diretório: lista de blobs e trees com nomes e permissões |
| **commit** | snapshot: aponta para uma tree + commit(s) pai + autor + mensagem |
| **tag** | referência nomeada e anotada para um commit |

```
commit 3a7f2c1
├── tree 9b4e8d0
│   ├── blob a1b2c3d  →  main.go
│   ├── blob f9e8d7c  →  go.mod
│   └── tree 2c3d4e5  →  pkg/
│       └── blob 6f7a8b9  →  handler.go
├── parent 1e2f3a4
├── author Felipe Soares <...>
└── message "feat: add health endpoint"
```

**Implicações práticas:**
- Dois arquivos com conteúdo idêntico compartilham o mesmo blob — Git não duplica
- Git não armazena diffs — armazena snapshots completos (mas comprime e agrupa em pack files)
- O SHA de um commit depende de toda a cadeia — mudar qualquer coisa gera um SHA diferente, por isso rebase "reescreve história"

### O Modelo das Três Árvores

Git gerencia **três "árvores"** simultaneamente — entender isso resolve 90% das confusões:

```
┌─────────────────┐    git add     ┌──────────────┐    git commit    ┌────────────┐
│  Working Tree   │ ─────────────► │    Index     │ ────────────────► │ Repository │
│  (disco local)  │                │  (Staging)   │                   │  (HEAD)    │
└─────────────────┘                └──────────────┘                   └────────────┘
         ▲                                                                   │
         └───────────────────────── git checkout / restore ─────────────────┘
```

| Árvore | O que é | Arquivo físico |
|---|---|---|
| **Working Tree** | seus arquivos no disco, o que você edita | arquivos normais |
| **Index (Staging Area)** | snapshot do próximo commit | `.git/index` |
| **HEAD** | o último commit do branch atual | `.git/HEAD` |

Isso explica comportamentos que parecem estranhos:
- `git diff` → compara **Working Tree vs Index**
- `git diff --cached` → compara **Index vs HEAD**
- `git diff HEAD` → compara **Working Tree vs HEAD** (ignora staging)
- `git status` → mostra diferenças entre as três árvores

### Referências (refs)

Refs são ponteiros para SHAs em `.git/refs/`:

```
.git/
├── HEAD                          → ref: refs/heads/main  (ou um SHA em detached)
├── refs/
│   ├── heads/
│   │   ├── main                  → SHA do commit mais recente do main
│   │   └── feature/auth          → SHA do commit mais recente da feature
│   ├── remotes/
│   │   └── origin/
│   │       ├── main              → estado do main no remoto (atualiza com fetch)
│   │       └── feature/auth
│   └── tags/
│       └── v1.2.0               → SHA de um commit ou de um objeto tag
└── packed-refs                   → refs compactadas (performance)
```

```bash
# Inspecionar diretamente
cat .git/HEAD
# ref: refs/heads/main

cat .git/refs/heads/main
# 3a7f2c1d8e9f4b2a...

# Detached HEAD: HEAD aponta para um SHA diretamente (não para um branch)
git checkout 3a7f2c1   # entra em detached HEAD — commits feitos aqui ficam "soltos"
```

### Como remotes funcionam

Um remote é **apenas uma URL com um nome**. Não é um servidor especial — é outro repositório Git que pode estar em outro diretório local, SSH, HTTP ou qualquer protocolo suportado.

```bash
# Ver remotes configurados
git remote -v
# origin  git@github.com:org/repo.git (fetch)
# origin  git@github.com:org/repo.git (push)

# O que acontece em cada operação:
# git fetch → baixa objetos e atualiza refs/remotes/* — NÃO toca o working tree
# git merge → integra o branch remoto no local — altera working tree
# git pull  → git fetch + git merge (ou rebase se configurado)
# git push  → envia objetos locais e atualiza o branch remoto
```

> [!warning] `fetch` vs `pull`
> `git fetch` é sempre seguro — só lê do remoto. `git pull` altera seu working tree. Em scripts e automações, prefira sempre `fetch` explícito seguido de `merge` ou `rebase` — isso dá controle sobre o que acontece.

---

## Comandos básicos — fundação indispensável

### Inicialização e clonagem

```bash
# Iniciar repositório do zero
git init meu-projeto
cd meu-projeto

# Iniciar no diretório atual
git init

# Clonar repositório existente
git clone https://github.com/org/repo.git
git clone git@github.com:org/repo.git          # via SSH (recomendado)
git clone https://github.com/org/repo.git meu-nome  # com nome de diretório diferente

# Clonar apenas o branch específico (útil para repos grandes)
git clone --branch develop --single-branch git@github.com:org/repo.git
```

### Verificando o estado

```bash
# Estado geral — o mais usado, rode sempre antes de fazer qualquer coisa
git status
git status -sb   # formato curto: M=modified, A=added, ??=untracked

# Ver o que mudou no working tree (antes do git add)
git diff

# Ver o que está staged (depois do git add, antes do commit)
git diff --cached
git diff --staged   # sinônimo

# Ver diferença de um arquivo específico
git diff -- arquivo.go

# Histórico de commits
git log
git log --oneline                         # uma linha por commit
git log --oneline --graph --all --decorate  # grafo visual de todos os branches
git log -5                                # últimos 5 commits
git log --author="Felipe"                 # filtrar por autor
git log --since="2 weeks ago"             # filtrar por data
git log --grep="feat:"                    # filtrar por mensagem
git log -- pkg/handler.go                 # histórico de um arquivo
```

### Staging e commit

```bash
# Adicionar arquivo ao staging
git add arquivo.go
git add pasta/                  # adicionar diretório inteiro
git add .                       # adicionar tudo (cuidado — use .gitignore corretamente)
git add *.go                    # por padrão glob
git add -p                      # modo interativo: escolher partes de cada arquivo (hunk)

# Remover do staging (sem apagar do disco)
git restore --staged arquivo.go
git reset HEAD arquivo.go       # equivalente, sintaxe antiga

# Criar commit
git commit -m "feat(api): adiciona endpoint de health check"
git commit                      # abre o editor configurado (melhor para mensagens longas)
git commit -am "fix: corrige validação"  # add + commit para arquivos já rastreados

# Alterar o último commit (antes do push)
git commit --amend -m "mensagem corrigida"
git commit --amend --no-edit    # alterar conteúdo mantendo a mensagem
```

> [!tip] `git add -p` — o comando mais subestimado
> Permite fazer staging de partes específicas de um arquivo (hunks), não o arquivo inteiro. Essencial para criar commits atômicos quando você alterou múltiplas coisas no mesmo arquivo. Digite `?` no modo interativo para ver as opções: `y` aceita, `n` rejeita, `s` divide o hunk, `e` edita manualmente.

### Branches

```bash
# Listar branches
git branch             # locais
git branch -r          # remotos
git branch -a          # todos
git branch -v          # com último commit

# Criar branch
git branch feature/auth

# Mudar de branch
git switch feature/auth          # sintaxe moderna (Git 2.23+)
git checkout feature/auth        # sintaxe clássica (ainda válida)

# Criar e já mudar para o branch
git switch -c feature/nova-rota
git checkout -b feature/nova-rota  # equivalente

# Criar branch a partir de um ponto específico
git switch -c hotfix/cors v1.2.0
git switch -c hotfix/cors 3a7f2c1

# Renomear branch atual
git branch -m novo-nome

# Deletar branch (só se já foi mergeado)
git branch -d feature/auth
git branch -D feature/auth   # forçar (cuidado — você perde os commits se não foram mergeados)

# Deletar branch remoto
git push origin --delete feature/auth
```

### Push e pull

```bash
# Push do branch atual para o remoto
git push origin feature/auth

# Push com tracking (cria rastreamento entre local e remoto)
git push -u origin feature/auth
# Depois desse -u, você pode só fazer `git push` sem especificar

# Atualizar referências do remoto sem alterar o working tree
git fetch origin
git fetch --all           # todos os remotes
git fetch --prune         # remover refs de branches que foram deletados no remoto

# Integrar mudanças do remoto
git pull                  # fetch + merge
git pull --rebase         # fetch + rebase (histórico mais limpo)
git pull origin main      # pull de branch específico

# Ver diferença entre local e remoto antes de integrar
git fetch origin
git diff origin/main..main        # o que tenho a mais
git diff main..origin/main        # o que o remoto tem a mais
git log origin/main..HEAD         # meus commits que ainda não foram
```

### Inspecionando o repositório

```bash
# Ver conteúdo de objetos internos
git cat-file -t 3a7f2c1    # tipo do objeto
git cat-file -p 3a7f2c1    # conteúdo do objeto
git cat-file -p HEAD        # commit atual
git cat-file -p HEAD^{tree} # árvore do commit atual

# Quem escreveu essa linha?
git blame pkg/handler.go
git blame -L 40,60 pkg/handler.go   # intervalo de linhas
git blame -w pkg/handler.go          # ignorar mudanças de whitespace

# Pesquisar dentro do repositório
git grep "func Handler"              # busca no working tree
git grep -n "TODO"                   # com número de linha
git grep "panic" -- "*.go"           # só arquivos .go

# SHA do HEAD
git rev-parse HEAD
git rev-parse --short HEAD   # SHA curto (7 chars)

# Mostrar informações de um commit
git show 3a7f2c1
git show HEAD~2              # dois commits atrás
git show v1.2.0:pkg/auth.go  # conteúdo de um arquivo em uma tag
```

### Desfazendo operações

```bash
# Descartar mudanças no working tree (IRREVERSÍVEL para arquivos não commitados)
git restore arquivo.go           # restaura para o que está no index
git restore .                    # restaurar tudo

# Remover do staging sem descartar mudanças
git restore --staged arquivo.go

# Desfazer o último commit mantendo mudanças no index
git reset --soft HEAD~1

# Desfazer o último commit mantendo mudanças no working tree (padrão)
git reset --mixed HEAD~1
git reset HEAD~1                 # equivalente

# Desfazer o último commit E as mudanças (DESTRUTIVO)
git reset --hard HEAD~1

# Criar commit que reverte um commit anterior (seguro para branches públicos)
git revert HEAD
git revert 3a7f2c1
git revert HEAD~3..HEAD          # reverter múltiplos commits

# Restaurar arquivo para versão de um commit anterior
git restore --source=HEAD~3 -- pkg/handler.go
git restore --source=v1.2.0 -- config.yaml
```

> [!warning] `reset --hard` é destrutivo
> Mudanças no working tree não commitadas são perdidas permanentemente. Antes de qualquer reset hard, verifique com `git status` e considere `git stash` para salvar o trabalho em progresso.

> [!tip] reflog é sua rede de segurança
> O reflog registra tudo que moveu HEAD nos últimos 90 dias (padrão). Quase nada em Git é permanentemente perdido enquanto o reflog existe.
> ```bash
> git reflog                  # lista todos os movimentos do HEAD
> git reset --hard HEAD@{3}   # voltar para um ponto anterior
> git checkout HEAD@{5}        # ir para um ponto sem alterar branches
> ```

---

## Na prática — fluxo completo do dia a dia

### Fluxo típico de feature

```bash
# 1. Partir do main atualizado
git switch main
git pull --rebase origin main

# 2. Criar branch de feature
git switch -c feature/adiciona-cache-redis

# 3. Trabalhar, fazendo commits atômicos
git add internal/cache/redis.go
git add internal/cache/redis_test.go
git commit -m "feat(cache): implementa cliente Redis com retry"

git add -p                   # staging seletivo de parte do arquivo
git commit -m "test(cache): adiciona testes de integração"

# 4. Manter a feature atualizada com main (rebase preferível)
git fetch origin
git rebase origin/main

# 5. Push e PR
git push -u origin feature/adiciona-cache-redis
```

### Fluxo de hotfix em produção

```bash
# 1. Criar hotfix a partir da tag em produção
git fetch --tags
git switch -c hotfix/cors-header v1.5.2

# 2. Corrigir
git add middleware/cors.go
git commit -m "fix(cors): adiciona header Access-Control-Allow-Origin"

# 3. Push e merge rápido
git push -u origin hotfix/cors-header

# 4. Após merge no main, taggear
git switch main && git pull
git tag -a v1.5.3 -m "Hotfix: corrige CORS header"
git push origin v1.5.3
```

### Inspecionando antes de integrar

```bash
# Antes de um git pull, ver o que está chegando
git fetch origin
git log HEAD..origin/main --oneline   # commits que virão
git diff HEAD..origin/main            # mudanças que virão

# Antes de um push, revisar o que está indo
git log origin/main..HEAD --oneline   # commits que vão
git diff origin/main..HEAD            # mudanças que vão
```

---

## Branches e merges

```bash
# Branch é só um arquivo com um SHA
cat .git/refs/heads/main

# Criar e mover para branch
git switch -c feature/auth

# Merge com commit explícito (preserva histórico)
git merge --no-ff feature/auth -m "feat: integra autenticação JWT"

# Fast-forward (só move o ponteiro, sem commit de merge)
git merge --ff-only hotfix/cors

# Abortar merge em conflito
git merge --abort

# Ver o que vai ser mergeado antes de fazer
git log main..feature/auth --oneline
git diff main...feature/auth         # três pontos: diff desde o ponto de divergência
```

## Rebase

Rebase reescreve commits — cria novos objetos com os mesmos diffs mas com pais diferentes. O resultado é um histórico linear mais legível.

```bash
# Rebase interativo: reordenar, squash, reword, drop
git rebase -i main

# Rebase de feature sobre main atualizado
git switch feature/cache
git rebase main

# Resolução de conflito durante rebase
git rebase main
# [conflito aparece]
git status                   # ver quais arquivos têm conflito
# editar arquivo, resolver conflito
git add arquivo.go
git rebase --continue

# Abortar e voltar ao estado anterior
git rebase --abort
```

> [!warning] Regra de ouro do rebase
> **Nunca rebase commits que já estão em um branch remoto compartilhado.** Rebase reescreve SHAs — outros desenvolvedores que basearam trabalho nesses commits terão divergência. Regra segura: rebase apenas em branches locais ainda não pushados, ou em feature branches que só você usa.

## Stash

```bash
# Salvar trabalho incompleto temporariamente
git stash push -m "WIP: refatorando auth middleware"

# Salvar incluindo arquivos untracked
git stash push -u -m "WIP: novo módulo de cache"

# Listar
git stash list

# Aplicar o mais recente e remover do stash
git stash pop

# Aplicar sem remover (pode aplicar várias vezes)
git stash apply stash@{2}

# Ver o conteúdo de um stash antes de aplicar
git stash show -p stash@{0}

# Criar branch a partir de um stash
git stash branch feature/experimento stash@{0}

# Descartar um stash
git stash drop stash@{1}
git stash clear   # descartar todos
```

## Tags

```bash
# Tag leve (só um ponteiro — não use para releases)
git tag v1.2.0

# Tag anotada (objeto próprio, com mensagem)
git tag -a v1.2.0 -m "Release 1.2.0 — suporte a múltiplos clusters"

# Push de tags (não vai automático com git push)
git push origin v1.2.0
git push origin --tags       # push de todas as tags

# Tag em commit específico
git tag -a v1.1.1 3a7f2c1 -m "Hotfix: corrige race condition"

# Listar e verificar
git tag                      # listar
git tag -l "v1.*"            # filtrar por padrão
git show v1.2.0              # detalhes da tag

# Deletar tag
git tag -d v1.2.0-rc1
git push origin --delete v1.2.0-rc1
```

---

## Casos de uso e boas práticas

### Estratégias de branching

**Trunk-Based Development (TBD)** — usado em times com CD maduro:
- Todo mundo commita direto em `main` (ou branches de vida curta < 2 dias)
- Feature flags escondem funcionalidades incompletas
- CI/CD roda em cada commit
- Requer boa cobertura de testes

**Git Flow** — para releases discretas (app mobile, biblioteca):
- `main` → produção estável
- `develop` → integração
- `feature/*`, `release/*`, `hotfix/*`
- Mais overhead, útil quando múltiplas versões em paralelo

**GitHub Flow** — meio-termo comum em SaaS:
- `main` sempre deployável
- Feature branches com PR
- Merge para main = deploy

> [!info] Métrica DORA
> Times de alta performance (Accelerate) usam TBD ou algo próximo. Lead time de mudança abaixo de 1 hora é atingível com TBD + CD. Git Flow tende a aumentar esse tempo estruturalmente.

### Commits semânticos (Conventional Commits)

```
<tipo>[escopo opcional]: <descrição curta>

[corpo opcional]

[rodapé(s) opcional(is)]
```

Tipos: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `ci`, `perf`, `build`

```bash
# Exemplos bons
feat(auth): implementa autenticação JWT com refresh token
fix(api): corrige race condition em pool de conexões
ci: adiciona cache de módulos Go no workflow de PR
refactor(handler): extrai lógica de validação para middleware
docs: atualiza README com instruções de setup local

# Exemplos ruins
git commit -m "fix"
git commit -m "wip"
git commit -m "ajustes"
git commit -m "funciona agora"
```

**Por que importa:** ferramentas como `semantic-release`, `git-cliff` e `release-please` geram CHANGELOGs e bump de versão automaticamente baseadas nesses tipos.

### Git Hooks

Hooks são scripts executados em pontos do ciclo de vida do Git. Ficam em `.git/hooks/` — mas para versionar, use **lefthook** ou **pre-commit**.

| Hook | Quando | Uso comum |
|---|---|---|
| `pre-commit` | antes de criar o commit | lint, format, secret scan |
| `commit-msg` | após digitar a mensagem | validar conventional commits |
| `pre-push` | antes do push | rodar testes, validar branch name |
| `post-merge` | após merge/pull | `npm install`, atualizar dependências |

```bash
# .git/hooks/pre-commit (exemplo básico)
#!/bin/bash
set -euo pipefail

go test ./...

if git diff --cached | grep -qE "(password|secret|token)\s*=\s*['\"][^'\"]{8,}"; then
  echo "ERRO: possível secret detectado. Use variáveis de ambiente."
  exit 1
fi
```

**Lefthook** (recomendado para times):
```yaml
# lefthook.yml
pre-commit:
  parallel: true
  commands:
    lint:
      run: golangci-lint run {staged_files}
    format:
      run: gofmt -l {staged_files}
    secrets:
      run: gitleaks detect --staged

commit-msg:
  commands:
    validate:
      run: npx commitlint --edit {1}
```

### .gitignore e .gitattributes

```bash
# .gitignore
*.log
*.tmp
/dist/
.env
.env.local
*.pem
*.key

# Verificar o que está sendo ignorado
git check-ignore -v arquivo.log

# Forçar rastrear arquivo mesmo que ignorado (raro)
git add -f arquivo-especial.log
```

```ini
# .gitattributes — normalização de line endings e diff
* text=auto eol=lf
*.go text eol=lf
*.sh text eol=lf
*.png binary
*.jpg binary
```

### Configuração profissional

```bash
# Identidade
git config --global user.name "Felipe Soares"
git config --global user.email "felipe@empresa.com"

# Editor padrão
git config --global core.editor "nvim"

# Pull strategy (evita merge commits acidentais)
git config --global pull.rebase true

# Push somente o branch atual
git config --global push.default current

# Diff com paginação colorida
git config --global core.pager "delta"

# Alias úteis
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.st "status -sb"
git config --global alias.ca "commit --amend --no-edit"
git config --global alias.unstage "restore --staged"

# Ver todas as configurações ativas
git config --list --show-origin
```

---

## Troubleshooting — cenários reais de produção

### Cenário 1: Push rejeitado após rebase local

```bash
$ git push origin feature/auth
! [rejected] feature/auth -> feature/auth (non-fast-forward)

# Diagnóstico
git log --oneline origin/feature/auth..HEAD   # commits locais a mais
git log --oneline HEAD..origin/feature/auth   # commits remotos a mais

# Se é um branch só seu e você sabe o que fez
git push --force-with-lease origin feature/auth
```

> [!warning] `--force-with-lease` vs `--force`
> `--force` sobrescreve cegamente. `--force-with-lease` verifica se o remoto ainda está no SHA que você conhecia — falha se alguém pushrou enquanto você trabalhava.

### Cenário 2: Merge acidental no main em produção

```bash
# Identificar o commit de merge problemático
git log --oneline --merges main | head -5

# Revert de um merge commit (precisa especificar o parent)
git revert -m 1 abc1234    # -m 1 = manter o lado main (parent 1)

git push origin main
```

### Cenário 3: Arquivo grande commitado acidentalmente

```bash
# Identificar o arquivo grande
git rev-list --objects --all | sort -k 2 | \
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
  sort -k3 -n -r | head -10

# Remover do histórico completo (REESCREVE HISTÓRICO)
git filter-repo --path secrets.pem --invert-paths

# Forçar push (coordinate com o time!)
git push --force-with-lease origin main
```

### Cenário 4: Conflito em arquivo binário

```bash
# Aceitar versão deles
git checkout --theirs -- imagem.png

# Aceitar nossa versão
git checkout --ours -- imagem.png

git add imagem.png
git merge --continue
```

### Cenário 5: Encontrar o commit que introduziu um bug

```bash
git bisect start
git bisect bad HEAD              # versão atual tem o bug
git bisect good v2.3.0           # essa versão estava boa

# Git faz checkout em commit intermediário — você testa
git bisect run go test ./...     # automatiza o teste

git bisect reset                 # volta para HEAD quando terminar
```

### Cenário 6: Recuperar commit "perdido" após reset --hard

```bash
git reflog                  # ver todo histórico de movimentos do HEAD
# HEAD@{0}: reset: moving to HEAD~3
# HEAD@{1}: commit: feat: adiciona endpoint de health check
# HEAD@{2}: commit: test: testes de integração

git reset --hard HEAD@{1}   # voltar para o commit "perdido"
# ou
git switch -c recuperacao HEAD@{1}  # criar branch com o conteúdo recuperado
```

---

## Nível avançado

### Git Internals: como o merge funciona

Git merge usa **three-way merge**: encontra o ancestral comum (merge base), compara as duas pontas e combina as mudanças. Se o mesmo trecho foi alterado nos dois lados → conflito.

```bash
# Encontrar merge base manualmente
git merge-base main feature/auth
# 1a2b3c4d...

# Ver exatamente o que cada lado mudou em relação ao ancestral
git diff 1a2b3c4d..main -- pkg/auth.go
git diff 1a2b3c4d..feature/auth -- pkg/auth.go
```

### Rerere (Reuse Recorded Resolution)

Se você resolve o mesmo conflito repetidamente durante rebase de um branch de longa duração:

```bash
git config --global rerere.enabled true
# Git grava a resolução e aplica automaticamente na próxima vez
```

### Worktrees

Múltiplos working trees do mesmo repositório sem clonar novamente:

```bash
# Criar worktree para hotfix em outro diretório
git worktree add ../projeto-hotfix hotfix/cors-fix

# Listar
git worktree list

# Remover quando terminar
git worktree remove ../projeto-hotfix
```

### Sparse Checkout

Para monorepos grandes — clonar e trabalhar apenas em subdiretórios:

```bash
git clone --filter=blob:none --sparse https://github.com/org/monorepo
cd monorepo
git sparse-checkout set services/auth services/gateway
```

### Signing de commits (SSH — mais simples que GPG)

```bash
git config --global gpg.format ssh
git config --global user.signingKey ~/.ssh/id_ed25519.pub
git config --global commit.gpgSign true

git log --show-signature   # verificar assinaturas
```

### Partial clone e shallow clone

```bash
# Shallow clone: apenas histórico recente (mais rápido em CI)
git clone --depth=1 https://github.com/org/repo

# Quando precisar do histórico completo depois
git fetch --unshallow

# Partial clone: sem baixar blobs (arquivos grandes)
git clone --filter=blob:none https://github.com/org/repo
```

### Git em pipelines CI/CD

```yaml
# GitHub Actions — boas práticas
- name: Checkout
  uses: actions/checkout@v4
  with:
    fetch-depth: 0          # necessário para git log, semantic-release
    fetch-tags: true

# Detectar qual serviço mudou em monorepo
- name: Changed files
  id: changes
  uses: dorny/paths-filter@v3
  with:
    filters: |
      auth:
        - 'services/auth/**'
      gateway:
        - 'services/gateway/**'

- name: Build auth
  if: steps.changes.outputs.auth == 'true'
  run: make build-auth
```

### Submodules vs subtree vs monorepo

| Abordagem | Quando usar | Trade-off |
|---|---|---|
| **Submodules** | dependência externa versionada explicitamente | complexidade no clone/update |
| **Subtree** | incorporar projeto externo no histórico | histórico poluído |
| **Monorepo** | times acoplados, dependências internas | CI/CD precisa de lógica de affected |
| **Polyrepo** | times desacoplados, deploy independente | coordenação entre repos |

```bash
# Submodule básico
git submodule add https://github.com/org/lib vendor/lib
git submodule update --init --recursive

# Clonar repositório com submodules
git clone --recurse-submodules https://github.com/org/projeto
```

---

## Referências

- **Pro Git** (livro gratuito) — git-scm.com/book — capítulos 10 (internals) e 7 (ferramentas avançadas)
- **Conventional Commits** — conventionalcommits.org
- **lefthook** — github.com/evilmartians/lefthook
- **git-filter-repo** — github.com/newren/git-filter-repo
- **Accelerate** (Forsgren, Humble, Kim) — métricas DORA e branching strategies
- **git-scm.com/docs** — referência oficial, `git help <comando>`
