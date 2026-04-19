---
tags:
  - exercicios
  - kubernetes
  - statefulset
  - headless-service
  - storage
  - databases
tipo: exercicios
area: kubernetes
conteudo: "[[05-Kubernetes/statefulset-headless]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Exercícios — StatefulSet e Headless Service

> Conteúdo: [[05-Kubernetes/statefulset-headless]] | Trilha: [[00-Trilha/kubernetes]]

**Infraestrutura:** k8s-master (192.168.3.30), worker1 (.31), worker2 (.32)
**Namespace de trabalho:** `estudo` (criar para estes exercícios)
**Namespace de observação:** `databases` (apenas leitura — nunca alterar)
**Atenção:** Nunca tocar em `databases/postgres-pvc`, `controle-gastos`, `promobot`.

---

## Exercício 1: Observar o StatefulSet de produção (leitura apenas)

**Contexto:** Antes de criar seu próprio StatefulSet, você vai analisar o postgres de produção para entender como um StatefulSet real está configurado.

**Missão:** Faça um inventário completo do StatefulSet `postgres` no namespace `databases` — sem alterar absolutamente nada.

**Requisitos:**
- [ ] Listar o StatefulSet: `kubectl get sts -n databases`
- [ ] Ver campos `READY`, `AGE`, `CONTAINERS`, `IMAGES`
- [ ] Fazer `kubectl describe sts postgres -n databases` e registrar:
  - `serviceName` (qual Headless Service está associado)
  - `podManagementPolicy`
  - `updateStrategy`
  - volumeClaimTemplates (StorageClass, tamanho, accessMode)
- [ ] Verificar os PVCs criados: `kubectl get pvc -n databases`
  - Qual é o nome do PVC? Segue o padrão `<template-name>-<sts-name>-<ordinal>`?
- [ ] Verificar o Headless Service: `kubectl get svc -n databases` — qual tem `ClusterIP: None`?
- [ ] Dentro do Pod `postgres-0`, verificar o hostname: `kubectl exec -n databases postgres-0 -- hostname`
- [ ] Verificar o `/etc/hosts` do Pod: `kubectl exec -n databases postgres-0 -- cat /etc/hosts`
- [ ] Documentar: como o Pod sabe seu próprio nome/ordinal?

**Verificação:**
```bash
kubectl get sts -n databases
kubectl get pvc -n databases
kubectl get svc -n databases | grep -v ClusterIP
# O Headless Service deve ter CLUSTER-IP = None
```

---

## Exercício 2: Criar Headless Service e entender o DNS

**Contexto:** Antes de criar o StatefulSet, você precisa criar o Headless Service — que fornece os registros DNS individuais por Pod. Você vai criar o Service e explorar como o DNS funciona.

**Missão:** Crie um Headless Service no namespace `estudo` e verifique a diferença entre um Service normal e um headless no DNS.

**Requisitos:**
- [ ] Criar namespace `estudo`
- [ ] Criar Service normal `postgres-normal` com `clusterIP` automático (não especificar o campo)
- [ ] Criar Headless Service `postgres` com `clusterIP: None`
- [ ] Verificar: `kubectl get svc -n estudo` — qual a diferença no campo `CLUSTER-IP`?
- [ ] Criar Pod temporário de DNS test: `kubectl run dns-test --image=busybox:1.36 -n estudo --restart=Never -- sleep 3600`
- [ ] Dentro do Pod, resolver os dois Services:
  - `nslookup postgres-normal.estudo.svc.cluster.local` → deve retornar um IP único (ClusterIP)
  - `nslookup postgres.estudo.svc.cluster.local` → deve retornar "can't resolve" ou nenhum registro (headless sem Pods ainda)
- [ ] Documentar a diferença no comportamento DNS
- [ ] Deletar apenas o Pod de teste (manter os Services para o próximo exercício)

**Verificação:**
```bash
kubectl get svc -n estudo
# postgres-normal → CLUSTER-IP = 10.x.x.x
# postgres        → CLUSTER-IP = None

kubectl exec -n estudo dns-test -- nslookup postgres-normal.estudo.svc.cluster.local
# Retorna um único IP (ClusterIP)

kubectl exec -n estudo dns-test -- nslookup postgres.estudo.svc.cluster.local
# Sem Pods ainda → pode não resolver ou resolver sem endereços
```

---

## Exercício 3: Primeiro StatefulSet com volumeClaimTemplates

**Contexto:** O time precisa de um PostgreSQL de desenvolvimento no namespace `estudo`. Você vai criar o StatefulSet completo com Headless Service, Secret e PVC automático.

**Missão:** Crie um StatefulSet PostgreSQL funcional com 1 réplica.

**Requisitos:**
- [ ] Criar Secret `postgres-secret` no namespace `estudo` com `username=estudo` e `password=EStudoPass123!`
- [ ] Confirmar que o Headless Service `postgres` do exercício anterior existe
- [ ] Criar StatefulSet `postgres` com:
  - `serviceName: postgres`
  - 1 réplica
  - imagem `postgres:16-alpine`
  - env vars a partir do Secret
  - `PGDATA=/var/lib/postgresql/data/pgdata` (subdir)
  - `volumeClaimTemplates` com `storageClassName: nfs-homelab`, `5Gi`, `RWO`
  - readinessProbe com `pg_isready`
- [ ] Monitorar a criação: `kubectl get pods -n estudo -w`
- [ ] Verificar que o PVC `data-postgres-0` foi criado automaticamente
- [ ] Confirmar que o Pod está `1/1 Running`
- [ ] Conectar ao PostgreSQL e criar uma tabela de teste:
  ```bash
  kubectl exec -n estudo postgres-0 -- psql -U estudo -d estudodb -c "CREATE TABLE teste (id serial, msg text);"
  kubectl exec -n estudo postgres-0 -- psql -U estudo -d estudodb -c "INSERT INTO teste (msg) VALUES ('dado persistente');"
  ```

**Verificação:**
```bash
kubectl get sts -n estudo
# READY: 1/1

kubectl get pvc -n estudo
# data-postgres-0   Bound   ...   5Gi   RWO   nfs-homelab

kubectl exec -n estudo postgres-0 -- psql -U estudo -d estudodb -c "SELECT * FROM teste;"
# Deve mostrar a linha inserida
```

---

## Exercício 4: Escalar e observar a ordem de criação

**Contexto:** O time quer adicionar réplicas de leitura. Você vai escalar o StatefulSet de 1 para 3 réplicas e observar que o Kubernetes respeita a ordem estrita.

**Missão:** Escale o StatefulSet para 3 réplicas e monitore a criação em ordem.

**Requisitos:**
- [ ] Abrir um terminal com `kubectl get pods -n estudo -w` para monitorar
- [ ] Em outro terminal: `kubectl scale sts postgres --replicas=3 -n estudo`
- [ ] Observar que `postgres-1` só é criado **após** `postgres-0` estar `1/1 Running`
- [ ] Observar que `postgres-2` só é criado **após** `postgres-1` estar `1/1 Running`
- [ ] Verificar que 3 PVCs foram criados: `data-postgres-0`, `data-postgres-1`, `data-postgres-2`
- [ ] Verificar os hostnames de cada Pod: `kubectl exec -n estudo postgres-<N> -- hostname`
- [ ] Verificar em quais nodes cada Pod foi agendado: `kubectl get pods -n estudo -o wide`
- [ ] Escalar de volta para 1 e observar que a deleção ocorre em **ordem inversa** (2 → 1)
- [ ] Verificar que os PVCs `data-postgres-1` e `data-postgres-2` ficaram órfãos após o scale-down

**Verificação:**
```bash
kubectl get pods -n estudo -w
# Ordem de criação: postgres-0 Running → postgres-1 Running → postgres-2 Running
# Ordem de deleção: postgres-2 Terminating → postgres-1 Terminating

kubectl get pvc -n estudo
# Após scale-down: 3 PVCs ainda existem mesmo com apenas 1 Pod
```

---

## Exercício 5: Identidade estável — Pod reiniciado monta o mesmo PVC

**Contexto:** A garantia central do StatefulSet é que `postgres-0` sempre monta `data-postgres-0`. Você vai deletar o Pod manualmente e confirmar que o Pod recriado monta exatamente o mesmo PVC — com os dados intactos.

**Missão:** Demonstre que a identidade estável funciona: dados persistem através de restarts.

**Requisitos:**
- [ ] Confirmar que `postgres-0` tem os dados da tabela `teste` (do exercício 3)
- [ ] Deletar o Pod `postgres-0`: `kubectl delete pod postgres-0 -n estudo`
- [ ] Observar que o StatefulSet recria `postgres-0` (mesmo nome)
- [ ] Aguardar `postgres-0` ficar `1/1 Running`
- [ ] Verificar que o **mesmo PVC** `data-postgres-0` foi montado (não um novo PVC)
- [ ] Conectar ao banco e confirmar que os dados ainda estão lá:
  ```bash
  kubectl exec -n estudo postgres-0 -- psql -U estudo -d estudodb -c "SELECT * FROM teste;"
  ```
- [ ] Verificar em qual node o Pod foi reescalonado — pode ser diferente do original?
- [ ] Documentar: o que aconteceria com os dados se estivéssemos usando `local-path` e o Pod fosse para outro node?

**Verificação:**
```bash
kubectl get pod postgres-0 -n estudo -o jsonpath='{.spec.nodeName}'
# Pode ter mudado de node (NFS permite isso — os dados seguem o Pod)

kubectl exec -n estudo postgres-0 -- psql -U estudo -d estudodb -c "SELECT * FROM teste;"
# Deve retornar: "dado persistente"
```

---

## Exercício 6: DNS por Pod — resolvendo instâncias individuais

**Contexto:** A réplica `postgres-1` precisa saber o endereço do primário `postgres-0` para configurar replicação. Você vai demonstrar como o DNS headless fornece endereços individuais por Pod.

**Missão:** Crie 3 réplicas e resolva o DNS individual de cada Pod a partir de dentro do cluster.

**Requisitos:**
- [ ] Escalar o StatefulSet para 3 réplicas (ou manter de exercícios anteriores)
- [ ] Criar Pod temporário: `kubectl run dns-test -n estudo --image=busybox:1.36 --restart=Never -- sleep 3600`
- [ ] Dentro do Pod de teste, resolver cada instância:
  - `nslookup postgres.estudo.svc.cluster.local` → deve retornar os 3 IPs dos Pods
  - `nslookup postgres-0.postgres.estudo.svc.cluster.local` → deve retornar apenas o IP de postgres-0
  - `nslookup postgres-1.postgres.estudo.svc.cluster.local` → IP de postgres-1
  - `nslookup postgres-2.postgres.estudo.svc.cluster.local` → IP de postgres-2
- [ ] Comparar os IPs retornados com `kubectl get pods -n estudo -o wide`
- [ ] Deletar `postgres-0` e aguardar recriar — o IP muda mas o DNS `postgres-0.postgres...` resolve o novo IP automaticamente
- [ ] Confirmar que após o restart, o DNS ainda resolve `postgres-0` para o novo IP
- [ ] Deletar o Pod de teste

**Verificação:**
```bash
kubectl exec -n estudo dns-test -- nslookup postgres-0.postgres.estudo.svc.cluster.local
# Deve retornar o IP atual do postgres-0

# Após deletar e recriar postgres-0:
kubectl exec -n estudo dns-test -- nslookup postgres-0.postgres.estudo.svc.cluster.local
# Deve retornar o NOVO IP — DNS atualizado automaticamente
```

---

## Exercício 7: Update com partition — canary de StatefulSet (avançado)

**Contexto:** O time quer fazer um upgrade do PostgreSQL 16 para 17, mas só quer atualizar a réplica de menor prioridade primeiro (canary) antes de tocar no primário.

**Missão:** Use o campo `partition` do updateStrategy para atualizar apenas `postgres-2`, validar, e depois promover o update para as demais réplicas.

**Requisitos:**
- [ ] Confirmar que o StatefulSet tem 3 réplicas rodando `postgres:16-alpine`
- [ ] Configurar `partition: 2` no updateStrategy:
  ```bash
  kubectl patch sts postgres -n estudo --type='json' \
    -p='[{"op":"replace","path":"/spec/updateStrategy/rollingUpdate/partition","value":2}]'
  ```
- [ ] Atualizar a imagem para `postgres:17-alpine`: `kubectl set image sts/postgres postgres=postgres:17-alpine -n estudo`
- [ ] Observar que apenas `postgres-2` é atualizado (ordinal >= 2) — `postgres-0` e `postgres-1` ficam em `postgres:16-alpine`
- [ ] Verificar as versões de cada Pod:
  ```bash
  kubectl get pods -n estudo -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.containers[0].image}{"\n"}{end}'
  ```
- [ ] Simular validação do canary: conectar no `postgres-2` e confirmar que funciona
- [ ] Promover o update: reduzir `partition` para `1`, aguardar `postgres-1` atualizar
- [ ] Reduzir `partition` para `0`, aguardar `postgres-0` atualizar
- [ ] Confirmar que todos os Pods estão em `postgres:17-alpine`

**Verificação:**
```bash
# Após setar partition=2 e mudar a imagem:
kubectl get pods -n estudo -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[0].image'
# postgres-0: postgres:16-alpine   ← não atualizado
# postgres-1: postgres:16-alpine   ← não atualizado
# postgres-2: postgres:17-alpine   ← atualizado (ordinal >= 2)
```

---

## Exercício 8: Lifecycle de PVCs — deletar StatefulSet e lidar com órfãos (Staff-level)

**Contexto:** O ambiente de estudo vai ser removido. Você precisa deletar o StatefulSet de forma controlada, entender o que acontece com os PVCs, e limpar completamente o namespace.

**Missão:** Execute o shutdown completo do ambiente e demonstre que os PVCs sobrevivem à deleção do StatefulSet.

**Requisitos:**
- [ ] Escalar o StatefulSet para 3 réplicas (se não estiver)
- [ ] Anotar quais PVCs existem: `kubectl get pvc -n estudo`
- [ ] Deletar o StatefulSet: `kubectl delete sts postgres -n estudo`
- [ ] Confirmar que os Pods foram deletados (em ordem inversa): `kubectl get pods -n estudo -w`
- [ ] Confirmar que os **PVCs não foram deletados**: `kubectl get pvc -n estudo`
  - `data-postgres-0`, `data-postgres-1`, `data-postgres-2` devem existir ainda
- [ ] Confirmar que os **PVs no NFS** ainda existem: `kubectl get pv | grep estudo`
- [ ] Criar um novo StatefulSet `postgres` com apenas 1 réplica — o que acontece com o PVC `data-postgres-0`?
  - O StatefulSet **reutiliza** o PVC existente (dados preservados!) ou cria um novo?
- [ ] Verificar que os dados da tabela `teste` ainda estão lá
- [ ] Limpeza final:
  ```bash
  kubectl delete sts postgres -n estudo
  kubectl delete pvc data-postgres-0 data-postgres-1 data-postgres-2 -n estudo
  kubectl delete secret postgres-secret -n estudo
  kubectl delete svc postgres postgres-normal -n estudo
  kubectl delete namespace estudo
  ```
- [ ] Confirmar que os PVs foram deletados (reclaimPolicy `Retain` vs `Delete`)

**Verificação:**
```bash
kubectl get pvc -n estudo
# Após deletar o StatefulSet: PVCs ainda existem

kubectl get pvc -n estudo
# Após recriar o StatefulSet com 1 réplica: data-postgres-0 em Bound (reusado!)

kubectl exec -n estudo postgres-0 -- psql -U estudo -d estudodb -c "SELECT * FROM teste;"
# Dados preservados da sessão anterior

kubectl get pv | grep estudo
# Após deletar PVCs com reclaimPolicy Delete: PVs desaparecem
# Após deletar PVCs com reclaimPolicy Retain: PVs ficam em Released
```

---

> [!tip] Ordem recomendada
> Exercícios 1 e 2 são preparatórios — observação e criação do Headless Service.
> Exercícios 3, 4 e 5 são sequenciais — cada um depende do anterior.
> Exercício 6 requer 3 réplicas — faça após o exercício 4.
> Exercícios 7 e 8 são avançados — faça após dominar os básicos.

> [!warning] Limpeza ao final
> O namespace `estudo` pode ter PVCs com reclaimPolicy `Retain` (nfs-homelab).
> Após todos os exercícios, verificar PVs órfãos: `kubectl get pv | grep Released`
> e deletar manualmente se necessário.
