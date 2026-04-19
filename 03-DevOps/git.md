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

Git é um sistema de controle de versão **distribuído** criado por Linus Torvalds em 2005 para gerenciar o código do kernel Linux. A palavra-chave é distribuído: cada clone é um repositório completo com todo o histórico — não existe dependência de um servidor central para a maioria das operações.

O modelo mental correto não é "servidor com backups locais". É uma rede de repositórios pares onde qualquer um pode ser fonte da verdade. O GitHub/GitLab é uma convenção social, não uma exigência técnica.

> [!info] Por que isso importa em DevOps
> Pipelines CI/CD rodam operações Git em cada push. Entender o modelo interno evita bugs silenciosos: por que o `git fetch` não atualiza o working tree? Por que `git pull --rebase` diverge do merge? Esses detalhes quebram pipelines.

---

## Como funciona internamente

### O Object Store

Git armazena tudo em `.git/objects/` como objetos imutáveis identificados por SHA-1 (migração para SHA-256 em andamento). Existem 4 tipos:

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

**Implicação prática:** dois arquivos com conteúdo idêntico compartilham o mesmo blob. Git não armazena diff — armazena snapshots completos (mas comprime e agrupa em pack files).

### Referências (refs)

Refs são ponteiros para SHAs em `.git/refs/`:
- `HEAD` → aponta para um branch ou diretamente para um commit (detached HEAD)
- `refs/heads/main` → SHA do commit mais recente do branch main
- `refs/remotes/origin/main` → cópia local do estado do remote
- `refs/tags/v1.0.0` → tag leve (SHA direto) ou objeto tag

```bash
cat .git/HEAD
# ref: refs/heads/main

cat .git/refs/heads/main
# 3a7f2c1d8e9f...
```

### O Index (Staging Area)

O index (`.git/index`) é a área de staging — o próximo commit que será criado. É um estado intermediário entre o working tree e o repositório:

```
working tree  →  (git add)  →  index  →  (git commit)  →  repository
```

Entender isso explica comportamentos que parecem estranhos:
- `git diff` compara working tree vs index
- `git diff --cached` compara index vs HEAD
- `git add -p` permite staging parcial de um arquivo

---

## Na prática — comandos e exemplos reais

### Inspecção do repositório

```bash
# Ver os objetos de um commit
git cat-file -p HEAD
git cat-file -p HEAD^{tree}

# Histórico com grafo
git log --oneline --graph --all --decorate

# O que mudou entre dois pontos
git diff main..feature/nova-rota
git diff main...feature/nova-rota   # três pontos: diff desde o ponto de divergência

# Quem mexeu nessa linha?
git blame -L 40,60 pkg/handler.go

# Quando esse bug foi introduzido?
git bisect start
git bisect bad HEAD
git bisect good v1.2.0
# git testa commits do meio até achar o culpado
```

### Branches e merges

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
```

### Rebase

Rebase reescreve commits — cria novos objetos com os mesmos diffs mas com pais diferentes.

```bash
# Rebase interativo: reordenar, squash, reword, drop
git rebase -i main

# Rebase de feature sobre main atualizado
git switch feature/cache
git rebase main

# Rebase e resolver conflito
git rebase main
# [conflito]
git add arquivo.go
git rebase --continue

# Abortar
git rebase --abort
```

> [!warning] Regra de ouro do rebase
> **Nunca rebase commits que já estão em um branch remoto compartilhado.** Rebase reescreve SHAs — outros desenvolvedores que basearam trabalho nesses commits terão divergência e precisarão fazer `git pull --force` ou resolver conflitos bizarros.
> 
> Regra segura: rebase apenas em branches locais ainda não pushados, ou em feature branches que só você usa.

### Stash

```bash
# Salvar trabalho incompleto temporariamente
git stash push -m "WIP: refatorando auth middleware"

# Listar
git stash list

# Aplicar o mais recente e remover do stash
git stash pop

# Aplicar sem remover (pode aplicar várias vezes)
git stash apply stash@{2}

# Criar branch a partir de um stash
git stash branch feature/experimento stash@{0}
```

### Tags

```bash
# Tag leve (só um ponteiro)
git tag v1.2.0

# Tag anotada (objeto próprio, com mensagem e assinatura opcional)
git tag -a v1.2.0 -m "Release 1.2.0 — suporte a múltiplos clusters"

# Push de tags (não vai automático com git push)
git push origin v1.2.0
git push origin --tags

# Tag em commit específico
git tag -a v1.1.1 3a7f2c1 -m "Hotfix: corrige race condition"
```

### Desfazendo coisas

```bash
# Desfazer commit mantendo as mudanças no index
git reset --soft HEAD~1

# Desfazer commit mantendo mudanças no working tree (mas fora do index)
git reset --mixed HEAD~1   # default

# Desfazer commit E as mudanças (DESTRUTIVO)
git reset --hard HEAD~1

# Revert: cria novo commit que desfaz (seguro para branches públicos)
git revert HEAD
git revert 3a7f2c1

# Restaurar arquivo específico para versão anterior
git restore --source=HEAD~3 -- pkg/handler.go

# Recuperar commit "perdido" após reset --hard
git reflog
git reset --hard HEAD@{3}
```

> [!tip] reflog é sua rede de segurança
> O reflog registra tudo que moveu HEAD nos últimos 90 dias (padrão). Quase nada em Git é permanentemente perdido enquanto o reflog existe. Antes de qualquer operação destrutiva, anote o SHA com `git rev-parse HEAD`.

### Configuração profissional

```bash
# Identidade
git config --global user.name "Felipe Soares"
git config --global user.email "felipe@empresa.com"

# Editor padrão
git config --global core.editor "nvim"

# Pull strategy padrão (evita merge commits acidentais)
git config --global pull.rebase true

# Push somente o branch atual
git config --global push.default current

# Diff com cores e paginação
git config --global core.pager "delta"

# Alias úteis
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.st "status -sb"
git config --global alias.ca "commit --amend --no-edit"
```

---

## Casos de uso e boas práticas

### Estratégias de branching

**Trunk-Based Development (TBD)** — usado em times com CD maduro:
- Todo mundo commita direto em `main` (ou branches de vida curta < 2 dias)
- Feature flags escondem funcionalidades incompletas
- CI/CD roda em cada commit
- Requer boa cobertura de testes e feature flags

**Git Flow** — para releases discretas (ex: app mobile, biblioteca):
- `main` → produção estável
- `develop` → integração
- `feature/*`, `release/*`, `hotfix/*`
- Mais overhead, útil quando múltiplas versões em paralelo

**GitHub Flow** — meio-termo comum:
- `main` sempre deployável
- Feature branches com PR
- Merge para main = deploy

> [!info] Métrica DORA
> Times de alta performance (segundo o livro Accelerate) usam TBD ou algo próximo. Tempo de lead time de mudança < 1 hora é atingível com TBD + CD. Git Flow tende a aumentar esse tempo.

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

# Exemplos ruins
git commit -m "fix"
git commit -m "wip"
git commit -m "ajustes"
```

**Por que importa:** ferramentas como `semantic-release`, `git-cliff` e `release-please` geram CHANGELOGs e bump de versão automaticamente baseadas nesses tipos.

### Git Hooks

Hooks são scripts executados em pontos do ciclo de vida do Git. Ficam em `.git/hooks/` — mas para versionar, use ferramentas como **pre-commit** ou **lefthook**.

```bash
# .git/hooks/pre-commit (exemplo básico)
#!/bin/bash
set -euo pipefail

# Não permite commit se testes falharem
go test ./...

# Não permite secrets óbvios
if git diff --cached | grep -qE "(password|secret|token)\s*=\s*['\"][^'\"]{8,}"; then
  echo "ERRO: possível secret detectado. Use variáveis de ambiente."
  exit 1
fi
```

**Hooks úteis:**
| Hook | Quando | Uso comum |
|---|---|---|
| `pre-commit` | antes de criar o commit | lint, format, secret scan |
| `commit-msg` | após digitar a mensagem | validar conventional commits |
| `pre-push` | antes do push | rodar testes, validar branch name |
| `post-merge` | após merge/pull | `npm install`, atualizar dependências |

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
# .gitignore — padrões por precedência
*.log
*.tmp
/dist/
/vendor/   # Go: commitar ou não depende do time
.env
.env.local
*.pem
*.key

# Verificar o que está sendo ignorado
git check-ignore -v arquivo.log

# Forçar rastrear arquivo mesmo que ignorado
git add -f arquivo-especial.log
```

```ini
# .gitattributes — normalização de line endings e diff
* text=auto eol=lf
*.go text eol=lf
*.sh text eol=lf
Dockerfile text eol=lf
*.png binary
*.jpg binary

# Diff customizado para arquivos específicos
*.md diff=markdown
Makefile diff=make
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
> `--force` sobrescreve cegamente. `--force-with-lease` verifica se o remoto ainda está no SHA que você conhecia — falha se alguém pushrou enquanto você trabalhava, evitando sobrescrever trabalho alheio.

### Cenário 2: Merge acidental no main em produção

```bash
# Identificar o commit de merge problemático
git log --oneline --merges main | head -5

# Revert de um merge commit (precisa especificar o parent)
git revert -m 1 abc1234    # -m 1 = manter o lado main (parent 1)

# Push imediato
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

### Cenário 5: Pipeline quebrou por causa de mensagem de commit

```bash
# commitlint falhando — reescrever a mensagem do último commit
git commit --amend -m "feat(api): adiciona endpoint de health check"

# Reescrever mensagem de commit mais antigo (interativo)
git rebase -i HEAD~3
# alterar 'pick' para 'reword' no commit desejado
```

### Cenário 6: Encontrar o commit que introduziu um bug

```bash
git bisect start
git bisect bad HEAD              # versão atual tem o bug
git bisect good v2.3.0           # essa versão estava boa

# Git faz checkout em commit intermediário
# Você testa manualmente ou roda um script
git bisect run go test ./...     # automatiza o teste

git bisect reset   # volta para HEAD quando terminar
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

Se você resolve o mesmo conflito repetidamente (ex: durante rebase de um branch de longa duração):

```bash
git config --global rerere.enabled true
# Git grava a resolução e aplica automaticamente na próxima vez
```

### Worktrees

Múltiplos working trees do mesmo repositório sem clonar novamente. Útil para trabalhar em hotfix sem perder o contexto atual:

```bash
# Criar worktree para hotfix em outro diretório
git worktree add ../projeto-hotfix hotfix/cors-fix

# Listar
git worktree list

# Remover quando terminar
git worktree remove ../projeto-hotfix
```

### Sparse Checkout

Para monorepos grandes: clonar e trabalhar apenas em subdiretórios:

```bash
git clone --filter=blob:none --sparse https://github.com/org/monorepo
cd monorepo
git sparse-checkout set services/auth services/gateway
```

### Signing de commits (GPG / SSH)

```bash
# Configurar assinatura com chave SSH (mais simples que GPG)
git config --global gpg.format ssh
git config --global user.signingKey ~/.ssh/id_ed25519.pub
git config --global commit.gpgSign true

# Verificar assinatura
git log --show-signature
```

### Git em pipelines CI/CD

```yaml
# GitHub Actions — boas práticas
- name: Checkout
  uses: actions/checkout@v4
  with:
    fetch-depth: 0          # necessário para git log, semantic-release, etc.
    fetch-tags: true        # pull tags explicitamente

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

### Partial clone e shallow clone

```bash
# Shallow clone: apenas histórico recente (mais rápido em CI)
git clone --depth=1 https://github.com/org/repo

# Problema: operações que precisam de histórico falham
# Solução em CI quando precisar do histórico completo:
git fetch --unshallow

# Partial clone: sem baixar blobs (arquivos grandes)
git clone --filter=blob:none https://github.com/org/repo
# Blobs são baixados on-demand
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
- **pre-commit** — pre-commit.com
- **lefthook** — github.com/evilmartians/lefthook
- **git-filter-repo** — github.com/newren/git-filter-repo
- **Accelerate** (Forsgren, Humble, Kim) — métricas DORA e branching strategies
- **git-scm.com/docs** — referência oficial, `git help <comando>`
