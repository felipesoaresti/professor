---
tags:
  - exercicios
  - devops
  - git
tipo: exercicios
area: devops
conteudo: "[[03-DevOps/git]]"
trilha: "[[00-Trilha/devops]]"
---

# Exercícios: Git

> Conteúdo: [[03-DevOps/git]] | Trilha: [[00-Trilha/devops]]

> [!warning] Sem respostas aqui
> Execute, observe o output, volte com o que você fez. A correção acontece no feedback.

---

## Exercício 1: Inspecionando o Object Store

**Contexto:** Você entrou num projeto novo e quer entender o que exatamente está no commit mais recente antes de qualquer mudança.

**Missão:** Usando o repositório em `/mnt/c/Users/felip/Documents/Professor` (que já é um repo git), inspecione a estrutura interna de um commit sem usar `git log`.

**Requisitos:**
- [ ] Exibir o conteúdo do objeto `HEAD` com `git cat-file`
- [ ] Exibir a tree apontada por esse commit
- [ ] Identificar o SHA de um blob específico (qualquer arquivo) a partir da tree
- [ ] Mostrar o conteúdo desse blob com `git cat-file -p <sha>`
- [ ] Confirmar que o conteúdo bate com o arquivo real em disco

**Verificação:**
```bash
git cat-file -t <sha>   # deve retornar o tipo correto do objeto
```

---

## Exercício 2: Staging Cirúrgico

**Contexto:** Você estava fazendo duas coisas ao mesmo tempo no mesmo arquivo: corrigiu um bug e começou uma refatoração. O time pediu que o bugfix vá pro main hoje e a refatoração amanhã, em PR separado.

**Missão:** Criar dois commits distintos a partir de mudanças no mesmo arquivo usando staging parcial.

**Requisitos:**
- [ ] Criar um arquivo de teste com pelo menos 30 linhas e duas seções de mudança distintas
- [ ] Adicionar as mudanças ao stage em partes usando `git add -p`
- [ ] Criar commit 1 com apenas a primeira parte
- [ ] Criar commit 2 com a segunda parte
- [ ] Verificar com `git diff HEAD~1 HEAD` e `git diff HEAD~2 HEAD~1` que cada commit contém apenas o esperado

**Verificação:**
```bash
git log --oneline -3
git show HEAD --stat
git show HEAD~1 --stat
```

---

## Exercício 3: Rebase Interativo

**Contexto:** Você trabalhou numa feature por 3 dias e fez 8 commits. Antes do PR, o tech lead pediu para "limpar o histórico" — ninguém precisa ver os commits de "fix typo" e "wip" intermediários.

**Missão:** Consolidar commits relacionados sem perder nenhuma mudança.

**Requisitos:**
- [ ] Criar um branch `feature/rebase-exercicio` com pelo menos 5 commits (pode ser arquivos de texto simples)
- [ ] Incluir pelo menos 1 commit com mensagem ruim ("fix", "wip", "ajuste")
- [ ] Usar `git rebase -i` para: fazer squash de commits relacionados, corrigir mensagem ruim com `reword`
- [ ] Resultado final: máximo 2 commits, ambos com mensagens no formato Conventional Commits
- [ ] Confirmar que o conteúdo final dos arquivos é idêntico ao anterior ao rebase

**Verificação:**
```bash
git log --oneline feature/rebase-exercicio
git diff main..feature/rebase-exercicio   # deve mostrar apenas as mudanças esperadas
```

---

## Exercício 4: Desfazendo sem destruir

**Contexto:** Um desenvolvedor fez `git commit -m "adiciona feature X"` e acabou de perceber que commitou por engano o arquivo `.env` com credenciais reais. O commit ainda não foi pushado.

**Missão:** Remover o arquivo do commit sem perder as outras mudanças commitadas.

**Requisitos:**
- [ ] Criar um commit que inclua um arquivo `.env` e pelo menos um arquivo legítimo
- [ ] Remover o `.env` do commit sem usar `git reset --hard` (não pode perder as outras mudanças)
- [ ] Confirmar que o arquivo `.env` ainda existe no working tree após a operação
- [ ] Confirmar que o arquivo legítimo continua no commit
- [ ] Adicionar `.env` ao `.gitignore` e commitar essa mudança

**Verificação:**
```bash
git show HEAD --name-only
git status
```

---

## Exercício 5: Git Bisect — Encontrando o commit do bug

**Contexto:** A aplicação funcionava na tag `v1.0` e está quebrada agora. São 15 commits de diferença. Você não quer revisar cada um manualmente.

**Missão:** Usar `git bisect` para identificar exatamente o commit que introduziu o problema.

**Requisitos:**
- [ ] Criar um script de teste simples (ex: verifica se um arquivo contém uma string específica — isso simula um teste de regressão)
- [ ] Criar uma sequência de pelo menos 8 commits, onde um deles "quebra" o comportamento
- [ ] Marcar o commit mais antigo como `good` e HEAD como `bad`
- [ ] Usar `git bisect run` com o script de teste para automatizar a busca
- [ ] Identificar o commit exato que introduziu o problema
- [ ] Executar `git bisect reset` e confirmar que está de volta ao HEAD

**Verificação:**
```bash
git bisect log   # antes do reset, mostra todo o processo
```

---

## Exercício 6: Hook de pre-commit

**Contexto:** O time está sofrendo com secrets sendo commitados acidentalmente. Você foi designado para implementar uma proteção local.

**Missão:** Criar um hook `pre-commit` que bloqueia commits com possíveis secrets.

**Requisitos:**
- [ ] O hook deve bloquear commit se qualquer arquivo staged contiver padrões como `password=`, `token=`, `secret=` seguidos de valor com 8+ caracteres
- [ ] O hook deve passar normalmente quando não há matches
- [ ] O hook deve exibir uma mensagem clara indicando qual arquivo e linha contém o problema
- [ ] O script deve usar `set -euo pipefail`
- [ ] Testar: criar arquivo com secret fake, tentar commitar, confirmar que foi bloqueado
- [ ] Testar: criar arquivo sem secret, confirmar que commit passa

**Verificação:**
```bash
echo 'password=minha_senha_secreta123' > teste-secret.txt
git add teste-secret.txt
git commit -m "test: isso deve falhar"
```

---

## Exercício 7: Worktree para hotfix paralelo

**Contexto:** Você está no meio de uma feature complexa (`feature/nova-auth`) com mudanças em 12 arquivos. Produção acabou de cair por causa de um bug crítico no main. Você precisa corrigir sem abandonar seu contexto.

**Missão:** Usar `git worktree` para criar um ambiente paralelo e fazer o hotfix sem tocar na feature em andamento.

**Requisitos:**
- [ ] Ter um branch `feature/worktree-demo` com pelo menos 3 arquivos modificados (não commitados não precisam estar aqui — use arquivos commitados)
- [ ] Criar um worktree apontando para `main` em `/tmp/hotfix-repo`
- [ ] No worktree, criar um branch `hotfix/exercicio`, fazer uma mudança e commitar
- [ ] Confirmar que o branch original `feature/worktree-demo` não foi afetado
- [ ] Remover o worktree após finalizar

**Verificação:**
```bash
git worktree list
ls /tmp/hotfix-repo
git log --oneline hotfix/exercicio
```

---

## Exercício 8: Conventional Commits + CHANGELOG automático

**Contexto:** O time quer automatizar a geração de CHANGELOG e bump de versão. Você vai configurar o tooling necessário.

**Missão:** Configurar `git-cliff` ou `conventional-changelog` para gerar CHANGELOG a partir do histórico de commits.

**Requisitos:**
- [ ] Instalar `git-cliff` (ou alternativa disponível no sistema)
- [ ] Criar pelo menos 6 commits seguindo Conventional Commits: 2 `feat`, 2 `fix`, 1 `docs`, 1 `refactor`
- [ ] Gerar um `CHANGELOG.md` automaticamente
- [ ] O CHANGELOG deve agrupar por tipo (Features, Bug Fixes, etc.)
- [ ] Confirmar que commits sem conventional format são ignorados ou agrupados em "Other"

**Verificação:**
```bash
cat CHANGELOG.md
git-cliff --version
```

---

## Exercício 9 (Staff-level): Remover arquivo sensível do histórico completo

**Contexto:** Um arquivo `credentials.json` foi commitado há 3 semanas e está em 15 commits. O repo está num fork privado, mas vai ser aberto. Você precisa limpar o histórico antes.

**Missão:** Usar `git filter-repo` para remover completamente o arquivo do histórico, verificar que não resta rastro, e preparar o repositório para o push limpo.

> [!warning] Impacto crítico
> Esse processo reescreve TODOS os SHAs após o commit contaminado. Em um repo real isso exige coordenação com todo o time (todos precisam re-clonar) e força push em todos os branches.

**Requisitos:**
- [ ] Criar um repositório de teste com pelo menos 10 commits, sendo que um deles adiciona `credentials.json` e commits subsequentes modificam outros arquivos
- [ ] Usar `git filter-repo --path credentials.json --invert-paths` para remover o arquivo
- [ ] Verificar com `git log --all --full-history -- credentials.json` que não há mais rastro
- [ ] Verificar que o conteúdo dos outros arquivos está intacto
- [ ] Executar `git reflog expire --expire=now --all && git gc --prune=now --aggressive` para limpar objetos órfãos
- [ ] Documentar (em comentário ou README) por que `--force-with-lease` não é suficiente nesse caso e quando usar `--force`

**Verificação:**
```bash
git log --all --full-history -- credentials.json   # deve retornar vazio
git count-objects -vH   # comparar antes/depois
```
