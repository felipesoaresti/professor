---
tags:
  - kubernetes
  - statefulset
  - headless-service
  - storage
  - dns
  - databases
area: kubernetes
tipo: conteudo
prerequisites:
  - "[[05-Kubernetes/volumes-storage]]"
  - "[[05-Kubernetes/secrets]]"
  - "[[05-Kubernetes/kubernetes-teoria-inicial]]"
next:
  - "[[11-Exercicios/statefulset-headless]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# StatefulSet e Headless Service

> Conteúdo: [[05-Kubernetes/statefulset-headless]] | Exercícios: [[11-Exercicios/statefulset-headless]] | Trilha: [[00-Trilha/kubernetes]]

Um Deployment trata todos os seus Pods como intercambiáveis — qualquer Pod pode substituir qualquer outro. Para bancos de dados, brokers de mensagens e sistemas distribuídos com estado, isso não funciona: cada instância tem sua própria identidade, seus próprios dados, e uma posição específica no cluster. O StatefulSet existe para isso.

---

## O que é e por que existe

### O problema do Deployment com estado

Um Deployment com PostgreSQL e 3 réplicas criaria 3 Pods com:
- Nomes aleatórios (`postgres-abc12`, `postgres-def34`)
- PVCs compartilhados ou cada um no storage local sem garantia de qual Pod vai para qual disco
- Sem ordem de inicialização — todos sobem simultaneamente
- Sem DNS individual por Pod — só o ClusterIP do Service

Isso quebra qualquer sistema de replicação: o réplica precisa saber quem é o primário, o primário precisa saber quem são os réplicas, e cada nó do cluster precisa de um endereço estável.

### As três garantias do StatefulSet

1. **Identidade estável de rede:** Pods têm nomes previsíveis e persistentes (`postgres-0`, `postgres-1`, `postgres-2`). O nome não muda nem após restart.

2. **Armazenamento estável:** cada Pod recebe seu próprio PVC via `volumeClaimTemplates`. `postgres-0` sempre monta o PVC `data-postgres-0`, mesmo após reiniciar ou ser reescalonado em outro node.

3. **Ordem de operações:** Pods são criados, escalados e deletados em ordem (`0 → 1 → 2`). O Pod `n` só é criado após o Pod `n-1` estar `Running` e `Ready`.

### Headless Service — a peça de rede

Um StatefulSet requer um Headless Service para fornecer endereços DNS individuais por Pod. `clusterIP: None` significa que o kube-proxy não cria regras de iptables para esse Service — em vez de um único ClusterIP, o DNS retorna diretamente os IPs dos Pods.

```
Service normal:   postgres-svc.default.svc.cluster.local → 10.96.0.50 (ClusterIP)
Headless Service: postgres.default.svc.cluster.local     → [10.244.1.5, 10.244.2.8, ...]

Pod individual (via Headless): postgres-0.postgres.default.svc.cluster.local → 10.244.1.5
                                postgres-1.postgres.default.svc.cluster.local → 10.244.2.8
```

O padrão `<pod-name>.<service-name>.<namespace>.svc.cluster.local` é o que permite que `postgres-1` saiba exatamente como conectar em `postgres-0` — mesmo que os IPs mudem.

---

## Como funciona internamente

### O StatefulSet Controller

Assim como o Deployment usa ReplicaSets como intermediário, o StatefulSet Controller gerencia os Pods diretamente. Não há ReplicaSet envolvido.

**Criação:** o Controller cria os Pods em ordem crescente, aguardando cada um ficar `Ready` antes de criar o próximo:
```
→ Cria postgres-0 → aguarda Ready
→ Cria postgres-1 → aguarda Ready
→ Cria postgres-2 → aguarda Ready
```

**Deleção:** a ordem é inversa (decrescente):
```
→ Deleta postgres-2 → aguarda terminar
→ Deleta postgres-1 → aguarda terminar
→ Deleta postgres-0 → aguarda terminar
```

**Restart/falha:** se `postgres-1` morre, o Controller recria `postgres-1` — mesmo nome, mesmo PVC, mesmo node se possível (via Node Affinity do PVC local-path). Não cria um Pod com nome diferente.

### volumeClaimTemplates — PVCs por réplica

```yaml
volumeClaimTemplates:
- metadata:
    name: data           # prefixo do PVC
  spec:
    storageClassName: nfs-homelab
    accessModes: [ReadWriteOnce]
    resources:
      requests:
        storage: 10Gi
```

Para um StatefulSet `postgres` com 3 réplicas, isso cria:
- `data-postgres-0` → montado em `postgres-0`
- `data-postgres-1` → montado em `postgres-1`
- `data-postgres-2` → montado em `postgres-2`

Os PVCs são **permanentes**: deletar o StatefulSet não deleta os PVCs. Você precisa deletá-los manualmente.

### DNS do Headless Service em detalhe

Quando um Pod é criado dentro de um StatefulSet com `serviceName: postgres`, o kubelet configura o hostname do Pod como `<pod-name>` e o subdomain como `<service-name>`. O CoreDNS cria automaticamente um registro A:

```
postgres-0.postgres.default.svc.cluster.local → IP do Pod postgres-0
postgres-1.postgres.default.svc.cluster.local → IP do Pod postgres-1
```

O registro do Service em si (`postgres.default.svc.cluster.local`) resolve para **todos** os IPs dos Pods — útil para discovery.

```bash
# Dentro de qualquer Pod no cluster:
nslookup postgres.default.svc.cluster.local
# Server: 10.96.0.10
# Non-authoritative answer:
# Name: postgres.default.svc.cluster.local
# Address: 10.244.1.5
# Address: 10.244.2.8

nslookup postgres-0.postgres.default.svc.cluster.local
# Address: 10.244.1.5   ← IP direto do postgres-0
```

### Update Strategy

**`RollingUpdate` (padrão):** atualiza em ordem reversa (`2 → 1 → 0`), aguardando cada Pod ficar Ready. O Pod `0` (primário, convencionalmente) é o último a ser atualizado.

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 2    # só atualiza Pods com índice >= 2 (canary por partition)
```

**`OnDelete`:** igual ao DaemonSet — só atualiza quando o Pod é deletado manualmente. Útil para upgrades manuais e controlados de bancos de dados.

### podManagementPolicy

```yaml
podManagementPolicy: OrderedReady    # padrão — ordem estrita
podManagementPolicy: Parallel        # todos sobem/descem simultaneamente
```

`Parallel` é útil quando os Pods são independentes (ex: Elasticsearch data nodes que não têm relação de primário/réplica entre si).

---

## Na prática — YAMLs e comandos

### StatefulSet completo — PostgreSQL de estudo

```yaml
# 1. Headless Service — OBRIGATÓRIO antes do StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: estudo
  labels:
    app: postgres
spec:
  clusterIP: None       # ← isso torna o Service headless
  selector:
    app: postgres
  ports:
  - name: postgres
    port: 5432
    targetPort: 5432
---
# 2. Service normal para acesso externo (opcional, mas útil)
apiVersion: v1
kind: Service
metadata:
  name: postgres-read
  namespace: estudo
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
---
# 3. StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: estudo
spec:
  serviceName: postgres         # deve casar com o nome do Headless Service
  replicas: 1                   # começar com 1 para estudo (aumentar depois)
  selector:
    matchLabels:
      app: postgres
  updateStrategy:
    type: RollingUpdate
  podManagementPolicy: OrderedReady
  template:
    metadata:
      labels:
        app: postgres
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_DB
          value: estudodb
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata   # subdir dentro do volume
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "$(POSTGRES_USER)", "-d", "$(POSTGRES_DB)"]
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 6
        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "$(POSTGRES_USER)"]
          periodSeconds: 15
          failureThreshold: 3
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      storageClassName: nfs-homelab
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 5Gi
```

```bash
# Secret necessário antes de aplicar
kubectl create secret generic postgres-secret \
  --from-literal=username=estudo \
  --from-literal=password='EStudoPass123!' \
  -n estudo
```

### Inspecionando o StatefulSet em produção (postgres existente)

```bash
# Observar apenas — sem alterar
kubectl get statefulset postgres -n databases
kubectl describe statefulset postgres -n databases | head -40

# Ver o PVC individual do postgres-0
kubectl get pvc -n databases
# data-postgres-0   Bound   pv-xxx   ...

# Ver os Pods e seus hostnames (sem alterar)
kubectl get pods -n databases -o wide
kubectl exec -n databases postgres-0 -- hostname
# postgres-0   ← nome estável

# Ver o DNS configurado dentro do Pod
kubectl exec -n databases postgres-0 -- cat /etc/hostname
kubectl exec -n databases postgres-0 -- cat /etc/hosts
```

### Comandos essenciais para StatefulSet

```bash
# Listar StatefulSets
kubectl get statefulset -A
kubectl get sts   # abreviação

# Escalar (cria em ordem: postgres-1, depois postgres-2)
kubectl scale sts postgres --replicas=3 -n estudo

# Escalar para baixo (deleta em ordem inversa: postgres-2, postgres-1)
kubectl scale sts postgres --replicas=1 -n estudo

# Rollout status (igual ao Deployment)
kubectl rollout status sts/postgres -n estudo

# Atualizar imagem
kubectl set image sts/postgres postgres=postgres:17-alpine -n estudo

# Rollback
kubectl rollout undo sts/postgres -n estudo

# Deletar StatefulSet SEM deletar os PVCs (padrão)
kubectl delete sts postgres -n estudo
kubectl get pvc -n estudo   # PVCs ainda existem!

# Deletar PVCs manualmente após confirmar que os dados não são mais necessários
kubectl delete pvc data-postgres-0 -n estudo
```

### Verificar DNS do Headless Service

```bash
# Criar Pod temporário para testar DNS
kubectl run dns-test --image=busybox:1.36 --restart=Never -n estudo -- \
  sh -c "nslookup postgres.estudo.svc.cluster.local && \
         nslookup postgres-0.postgres.estudo.svc.cluster.local"

kubectl logs dns-test -n estudo
kubectl delete pod dns-test -n estudo
```

---

## Casos de uso e boas práticas

### Quando usar StatefulSet vs Deployment

| Critério | Deployment | StatefulSet |
|---|---|---|
| Identidade do Pod importa? | Não | Sim |
| Cada réplica tem dados próprios? | Não | Sim |
| Precisa de DNS por Pod? | Não | Sim |
| Ordem de inicialização importa? | Não | Sim |
| Exemplos | API REST, nginx, workers | PostgreSQL, Redis Cluster, Kafka, Zookeeper, etcd |

### Padrão: Primary + Réplicas com Headless Service

Para um PostgreSQL com primário e réplicas de leitura:

```
postgres-0  → primário (leitura + escrita)
postgres-1  → réplica (leitura)
postgres-2  → réplica (leitura)
```

A réplica `postgres-1` se conecta ao primário via:
```
postgres-0.postgres.<namespace>.svc.cluster.local:5432
```

Não precisa de descoberta dinâmica — o nome DNS é sempre o mesmo, independente de qual IP o Pod tem no momento.

### StorageClass para StatefulSets de produção

- **NFS com Retain:** `nfs-homelab` — dados preservados mesmo se o PVC for deletado acidentalmente. Correto para produção.
- **local-path:** rápido, mas amarra o Pod ao node onde o PVC foi criado. Se o node morrer, o Pod fica Pending. Evitar para dados críticos.
- **Nunca usar emptyDir em StatefulSet:** perde tudo ao reiniciar.

### `PGDATA` e o problema do "diretório não vazio"

O PostgreSQL recusa inicializar se o diretório `PGDATA` não estiver vazio ou não tiver sido inicializado por ele. Quando o PVC é montado em `/var/lib/postgresql/data`, o diretório root do mount pode ter arquivos de metadados do filesystem. A solução é usar um subdiretório:

```yaml
env:
- name: PGDATA
  value: /var/lib/postgresql/data/pgdata   # subdir limpo dentro do volume
```

### Nunca deletar PVCs de produção sem backup

```bash
# SEMPRE fazer backup antes de deletar PVCs de StatefulSets
# Para o postgres do homelab: NUNCA deletar
# Para estudo: sempre verificar duas vezes o namespace antes de deletar
kubectl delete pvc data-postgres-0 -n estudo    # OK — namespace de estudo
kubectl delete pvc data-postgres-0 -n databases # NUNCA
```

---

## Troubleshooting — cenários reais de produção

### Cenário 1: Pod do StatefulSet travado em Pending

```bash
kubectl get pods -n estudo
# postgres-1   0/1   Pending   10m

kubectl describe pod postgres-1 -n estudo | grep -A5 Events
# Warning  FailedScheduling  ... persistentvolumeclaim "data-postgres-1" not found

# O PVC não foi criado — normalmente o StatefulSet cria automaticamente
# Verificar se o volumeClaimTemplate está correto no spec
kubectl get sts postgres -n estudo -o jsonpath='{.spec.volumeClaimTemplates}'

# Verificar se a StorageClass existe
kubectl get sc nfs-homelab

# Se a StorageClass não existir ou o provisioner não estiver rodando:
kubectl get pods -n kube-system | grep csi
```

### Cenário 2: StatefulSet stuck — Pod `n` não sobe porque Pod `n-1` não está Ready

```bash
# postgres-0 está Running mas not Ready → postgres-1 nunca é criado
kubectl get pods -n estudo
# postgres-0   0/1   Running   5m   ← not Ready

# Verificar por que postgres-0 não está Ready
kubectl describe pod postgres-0 -n estudo | grep -A10 Events
# Readiness probe failing

# Verificar logs
kubectl logs postgres-0 -n estudo
# FATAL: data directory "/var/lib/postgresql/data" has wrong ownership
# → PGDATA não está configurado corretamente (diretório com permissão errada)
```

### Cenário 3: Pod reiniciado, PVC ligado ao node errado (local-path)

```bash
# postgres-0 foi reiniciado e não está conseguindo montar o PVC
kubectl describe pod postgres-0 | grep Events
# Warning  FailedAttachVolume  ... node "k8s-worker2" does not have volume...
# O PVC local-path está no worker1, mas o Pod foi reescalonado no worker2

# Local-path tem Node Affinity no PV:
kubectl describe pv <pv-name> | grep -A5 "Node Affinity"
# kubernetes.io/hostname=k8s-worker1

# Solução: forçar o Pod a rodar no node correto via nodeSelector
# ou migrar para NFS (sem afinidade de node)

# Verificar em qual node o Pod precisa estar
kubectl get pod postgres-0 -o jsonpath='{.spec.nodeName}'
```

### Cenário 4: Escalar para baixo — garantir que todos os dados foram sincronizados

```bash
# Antes de escalar para baixo um PostgreSQL com réplicas:
# 1. Verificar lag de replicação
kubectl exec -n estudo postgres-0 -- psql -U estudo -c \
  "SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn FROM pg_stat_replication;"

# 2. Escalar para baixo (vai deletar em ordem inversa)
kubectl scale sts postgres --replicas=1 -n estudo

# 3. Os PVCs das réplicas ficam — verificar
kubectl get pvc -n estudo
# data-postgres-0   Bound
# data-postgres-1   Bound   ← PVC órfão (sem Pod)
# data-postgres-2   Bound   ← PVC órfão (sem Pod)

# 4. Se não precisar mais dos dados, deletar os PVCs órfãos
kubectl delete pvc data-postgres-1 data-postgres-2 -n estudo
```

### Cenário 5: PVC órfão bloqueia recriação do StatefulSet

```bash
# Deletou o StatefulSet mas os PVCs ainda existem
# Ao recriar o StatefulSet, ele vai tentar usar os PVCs existentes (correto!)
# Mas se o PVC tem um PV em Released/Wrong state:

kubectl get pvc data-postgres-0 -n estudo
# STATUS: Lost   ← PV foi deletado mas PVC ainda existe

kubectl describe pvc data-postgres-0 -n estudo
# Status: Lost
# Volume: pv-xxx (does not exist)

# Solução: deletar o PVC e deixar o StatefulSet criar um novo
kubectl delete pvc data-postgres-0 -n estudo
# Na próxima vez que o StatefulSet criar postgres-0, um novo PVC é criado
```

---

## Nível avançado — CKA/Staff

### O ordinal e a identidade do Pod

O nome `<statefulset-name>-<ordinal>` é a identidade estável. O ordinal começa em 0 por padrão (K8s 1.26+ permite mudar com `spec.ordinals.start`).

O Pod sabe seu próprio ordinal via hostname:
```bash
kubectl exec postgres-1 -- hostname
# postgres-1

# Em scripts de inicialização, extrair o ordinal:
ORDINAL=$(hostname | awk -F'-' '{print $NF}')
# ORDINAL=1
# Se ORDINAL == 0 → sou o primário
# Se ORDINAL > 0 → sou réplica, conectar em postgres-0.postgres.<ns>.svc.cluster.local
```

### Partition — canary update de StatefulSet

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 2   # apenas Pods com ordinal >= 2 são atualizados
```

Com `partition: 2` em um StatefulSet de 3 réplicas: apenas `postgres-2` é atualizado quando a imagem muda. `postgres-0` e `postgres-1` permanecem na versão antiga. Para promover a atualização, reduzir a partition gradualmente (`2 → 1 → 0`).

### Headless Service com `publishNotReadyAddresses`

```yaml
spec:
  clusterIP: None
  publishNotReadyAddresses: true   # inclui Pods não-Ready nos registros DNS
```

Útil durante a inicialização: o `postgres-0` precisa anunciar sua presença no DNS antes de estar completamente Ready (ex: para que o Patroni/pg_rewind consiga conectar durante a recuperação).

### StatefulSet com init container para setup de replicação

```yaml
initContainers:
- name: setup-replica
  image: postgres:16-alpine
  command:
  - /bin/bash
  - -c
  - |
    ORDINAL=$(hostname | awk -F'-' '{print $NF}')
    if [ "$ORDINAL" -ne 0 ]; then
      # Sou réplica — clonar dados do primário
      pg_basebackup -h postgres-0.postgres.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local \
        -U replicator -D /var/lib/postgresql/data/pgdata -R -P
    fi
  volumeMounts:
  - name: data
    mountPath: /var/lib/postgresql/data
```

### Inspecionar o StatefulSet do postgres de produção (sem alterar)

```bash
# Ver a configuração completa do postgres de produção para referência
kubectl get sts postgres -n databases -o yaml | head -80

# Ver os PVCs associados
kubectl get pvc -n databases

# Ver os eventos recentes
kubectl describe sts postgres -n databases | tail -20

# Ver o Service headless associado
kubectl get svc -n databases
kubectl describe svc postgres -n databases | grep -E "Type|ClusterIP|Selector"
# Type: ClusterIP
# ClusterIP: None   ← headless
```

---

## Referências

- **Kubernetes docs** — [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- **Kubernetes docs** — [Headless Services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)
- **Kubernetes docs** — [Stable Network ID](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#stable-network-id)
- **PostgreSQL on Kubernetes** — cloudnativepg.io (operador de referência para PostgreSQL em K8s)
- **Patroni** — github.com/zalando/patroni (HA para PostgreSQL em StatefulSet)
- **Linuxtips DK8S** — `/mnt/c/Users/felip/Documents/Kubernets/DK8S`
- `kubectl explain statefulset.spec.volumeClaimTemplates`
- `kubectl explain statefulset.spec.updateStrategy`
