---
tags:
  - kubernetes
  - volumes
  - storage
  - pv
  - pvc
  - storageclass
area: kubernetes
tipo: conteudo
prerequisites:
  - "[[05-Kubernetes/kubernetes-teoria-inicial]]"
  - "[[05-Kubernetes/daemonset]]"
next:
  - "[[11-Exercicios/volumes-storage]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Volumes em Kubernetes — StorageClass, PV e PVC

> Conteúdo: [[05-Kubernetes/volumes-storage]] | Exercícios: [[11-Exercicios/volumes-storage]] | Trilha: [[00-Trilha/kubernetes]]

Containers são efêmeros — quando morrem, levam o filesystem junto. Volumes resolvem isso. O Kubernetes tem três camadas de abstração para storage persistente: **StorageClass** (como provisionar), **PersistentVolume** (o storage provisionado), e **PersistentVolumeClaim** (a requisição de storage pelo workload). Cada camada tem um papel distinto e um owner diferente.

---

## O que é e por que existe

### O problema

Um container escreve dados em `/data`. O Pod é reiniciado por OOM kill → o container sobe do zero → `/data` está vazio. Ou pior: você tem 3 réplicas de um banco de dados e cada uma enxerga um filesystem diferente.

Volumes resolvem persistência. Mas volumes simples (`emptyDir`, `hostPath`) são amarrados ao Pod ou ao node — não são portáveis. Para storage que sobrevive a restarts, migrações de Pod entre nodes, e deleções de Pod, você precisa de **PersistentVolumes**.

### A separação de responsabilidades

```
Administrador do cluster          →  StorageClass (como provisionar)
Administrador do cluster          →  PersistentVolume (storage provisionado) [static]
Provisioner (automático)          →  PersistentVolume (storage provisionado) [dynamic]
Desenvolvedor / time de app       →  PersistentVolumeClaim (quanto storage preciso)
Pod                               →  monta o PVC como volume
```

Essa separação permite que o time de app declare "preciso de 10Gi de storage com acesso RWO" sem saber se o storage vem de NFS, disco local, EBS, ou Ceph.

---

## Como funciona internamente

### O fluxo completo

```
StorageClass define:
  - provisioner (quem cria o PV)
  - reclaimPolicy (o que fazer quando o PVC é deletado)
  - volumeBindingMode (quando fazer o bind)
  - parameters (opções específicas do provisioner)

       ┌─ Static: admin cria PV manualmente
PV ←──┤
       └─ Dynamic: provisioner cria PV automaticamente quando PVC é criado

PVC requisita:
  - storageClassName (qual StorageClass usar)
  - accessModes (RWO / ROX / RWX / RWOP)
  - storage (quanto precisa)

Binding: o Controller encontra um PV compatível e liga ao PVC (1:1)

Pod monta o PVC como volume → acessa o storage
```

### Ciclo de vida do PV

```
Available  →  (PVC cria e casa) →  Bound
Bound      →  (PVC deletado)    →  Released
Released   →  (reclaimPolicy)   →  Retained (fica Released, dados preservados)
                                   Deleted   (PV e dados deletados)
                                   Recycled  (deprecated — não usar)
```

> [!warning] PV Released não pode ser re-bound automaticamente
> Quando a reclaimPolicy é `Retain` e o PVC é deletado, o PV fica em estado `Released`. Ele tem os dados do PVC anterior mas não pode ser re-usado por um novo PVC automaticamente. É necessário remover a referência ao PVC antigo do `.spec.claimRef` do PV para que ele volte ao estado `Available`.

### Access Modes — o que cada um significa

| Mode | Abreviação | Significado |
|---|---|---|
| `ReadWriteOnce` | RWO | Montado como read-write por **um único node** |
| `ReadOnlyMany` | ROX | Montado como read-only por **múltiplos nodes** |
| `ReadWriteMany` | RWX | Montado como read-write por **múltiplos nodes** |
| `ReadWriteOncePod` | RWOP | Montado como read-write por **um único Pod** (K8s 1.22+) |

> [!info] Access mode ≠ controle de acesso real
> O access mode é uma promessa ao provisioner, não uma garantia de enforcement. Um PV com RWO montado em dois Pods no mesmo node pode resultar em corrupção de dados — o Kubernetes não bloqueia, apenas o sistema de arquivos subjacente (ou não) aplica exclusão.

**NFS suporta RWX.** Discos locais (`local-path`, `hostPath`) suportam apenas RWO — um disco local pertence a um node.

### VolumeBindingMode

**`Immediate`:** o PV é provisionado e o PVC é bound assim que o PVC é criado, antes de qualquer Pod ser criado. Problema: para storage local (que existe em um node específico), o PV pode ser criado em um node diferente do que o Pod vai rodar → Pod fica Pending.

**`WaitForFirstConsumer`:** o PV é provisionado apenas quando um Pod que usa o PVC é criado e agendado. O provisioner sabe em qual node o Pod vai rodar e cria o PV naquele node. Correto para storage local.

```
local-path StorageClass no homelab → WaitForFirstConsumer (correto)
nfs-homelab StorageClass           → Immediate (correto — NFS não tem afinidade de node)
```

### StorageClasses do homelab

```bash
kubectl get storageclass
```

| StorageClass | Provisioner | ReclaimPolicy | BindingMode |
|---|---|---|---|
| `local-path` | `rancher.io/local-path` | Delete | WaitForFirstConsumer |
| `nfs-homelab` | `nfs.csi.k8s.io` | **Retain** | Immediate |
| `nfs-homelab-delete` | `nfs.csi.k8s.io` | Delete | Immediate |

- **`local-path`**: cria um diretório em `/opt/local-path-provisioner/` no node onde o Pod roda. Rápido, sem dependências externas. Dados perdidos se o node morrer. Bom para desenvolvimento e caches.
- **`nfs-homelab`**: usa o servidor NFS em `192.168.3.11:/mnt/nfs-data/k8s`. Dados persistem independente do node. `Retain` = dados preservados quando o PVC é deletado. Para dados importantes.
- **`nfs-homelab-delete`**: igual ao anterior mas `Delete` = dados deletados com o PVC. Para dados temporários que precisam de RWX.

---

## Na prática — YAMLs e comandos

### PVC simples com local-path

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: meu-pvc
  namespace: default
spec:
  storageClassName: local-path
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
# Criar e verificar
kubectl apply -f pvc.yaml
kubectl get pvc meu-pvc
# NAME       STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# meu-pvc    Pending   ...      ...        RWO            local-path     5s
# STATUS = Pending porque local-path usa WaitForFirstConsumer
# Só vai para Bound quando um Pod montar o PVC

kubectl describe pvc meu-pvc
# Events: "waiting for first consumer to be created before binding"
```

### Pod montando o PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: default
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c", "while true; do date >> /data/log.txt; sleep 5; done"]
    volumeMounts:
    - name: dados
      mountPath: /data
  volumes:
  - name: dados
    persistentVolumeClaim:
      claimName: meu-pvc    # referencia o PVC pelo nome
```

```bash
# Após criar o Pod, o PVC vai para Bound
kubectl get pvc meu-pvc
# STATUS: Bound
# VOLUME: pvc-xxxxxxxx-... (PV criado dinamicamente)

# Ver o PV criado automaticamente
kubectl get pv
kubectl describe pv <nome-do-pv>
# Source → local-path → path no node onde o Pod está rodando

# Confirmar que os dados persistem após reiniciar o Pod
kubectl exec app -- cat /data/log.txt
kubectl delete pod app
kubectl apply -f pod.yaml   # recria o Pod
kubectl exec app -- cat /data/log.txt   # dados ainda lá!
```

### PVC com NFS (RWX — múltiplos Pods)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dados-compartilhados
  namespace: default
spec:
  storageClassName: nfs-homelab-delete
  accessModes:
  - ReadWriteMany     # múltiplos Pods podem escrever
  resources:
    requests:
      storage: 5Gi
```

```yaml
# Deployment com múltiplas réplicas compartilhando o mesmo PVC
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: nginx:1.27
        volumeMounts:
        - name: shared
          mountPath: /usr/share/nginx/html
      volumes:
      - name: shared
        persistentVolumeClaim:
          claimName: dados-compartilhados
```

> [!warning] RWX com NFS e escrita simultânea
> NFS suporta RWX, mas não é um filesystem de cluster como Ceph/GlusterFS. Múltiplos escritores simultâneos no mesmo arquivo podem causar corrupção. Use RWX para leitura compartilhada ou quando só um Pod escreve de cada vez.

### PV estático (sem StorageClass, sem provisionamento dinâmico)

```yaml
# Administrador cria o PV manualmente — aponta para NFS diretamente
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-manual
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.3.11
    path: /mnt/nfs-data/k8s/manual-pv
---
# PVC que se liga a esse PV específico (via storageClassName vazia + seletor)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-manual
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: ""   # sem StorageClass = binding estático
  resources:
    requests:
      storage: 10Gi
  volumeName: pv-nfs-manual   # bind direto pelo nome do PV
```

### Volumes não-persistentes úteis

**`emptyDir`** — criado quando o Pod inicia, deletado quando o Pod termina. Compartilhado entre containers do mesmo Pod.

```yaml
volumes:
- name: cache
  emptyDir:
    medium: Memory    # opcional: usar RAM (tmpfs) em vez de disco
    sizeLimit: 256Mi
```

**`configMap` e `secret` como volume** — montar ConfigMaps e Secrets como arquivos.

```yaml
volumes:
- name: config
  configMap:
    name: meu-configmap
- name: creds
  secret:
    secretName: meu-secret
    defaultMode: 0400   # permissão dos arquivos gerados
```

**`hostPath`** — monta diretório do node (visto em DaemonSets). Não recomendado para apps normais.

### Comandos essenciais de storage

```bash
# Listar StorageClasses
kubectl get storageclass
kubectl get sc   # abreviação

# Ver qual é a default (marcada com "(default)")
kubectl get sc
# local-path (default)  rancher.io/local-path  Delete  WaitForFirstConsumer

# Listar PVs (cluster-scoped, sem namespace)
kubectl get pv
kubectl get pv -o wide   # mostra STORAGECLASS e REASON

# Listar PVCs (namespace-scoped)
kubectl get pvc -A        # todos os namespaces
kubectl get pvc -n default

# Detalhes de um PV
kubectl describe pv <nome>

# Detalhes de um PVC — mostra o PV bound e eventos
kubectl describe pvc <nome>

# Ver onde o local-path armazena os dados no node
kubectl describe pv <pv-local-path> | grep Path
# /opt/local-path-provisioner/pvc-xxx.../default_meu-pvc

# Verificar uso de disco de um PVC via Pod
kubectl exec <pod> -- df -h /data
```

---

## Casos de uso e boas práticas

### Quando usar cada StorageClass do homelab

| Situação | StorageClass | Por quê |
|---|---|---|
| App stateless com cache local | `local-path` | Rápido, sem overhead de rede |
| Banco de dados (PostgreSQL, Redis) | `nfs-homelab` | Dados persistem, Retain protege contra deleção acidental |
| Assets compartilhados entre Pods | `nfs-homelab-delete` | RWX — múltiplos leitores/escritores |
| Job temporário que processa arquivos | `nfs-homelab-delete` | Delete = limpeza automática |
| Dados críticos que precisam sobreviver ao PVC | `nfs-homelab` | Retain = dados ficam no NFS mesmo após PVC deletado |

### Sempre defina `resources.requests.storage` com folga

O PVC requisita um mínimo. O PV provisionado pode ser maior (depende do provisioner). Mas você não pode aumentar um PVC local-path depois de criado sem recriar. Para NFS, é possível expandir se o StorageClass tem `allowVolumeExpansion: true`.

```bash
# Verificar se a StorageClass permite expansão
kubectl describe sc nfs-homelab | grep AllowVolume
```

### Reclame Policy — a decisão mais importante

- **`Delete`**: PVC deletado → PV deletado → dados deletados. Cuidado em produção.
- **`Retain`**: PVC deletado → PV fica em `Released` com dados intactos. Você decide o que fazer depois.

Para dados de banco em produção: **sempre `Retain`**. Você pode sempre deletar manualmente. Não pode recuperar dados deletados automaticamente.

### Não use `hostPath` em Deployments

`hostPath` amarra o Pod a um node específico. Se o node cair, o Pod não pode ser reescalonado em outro node com acesso aos dados. Use PVC + StorageClass.

---

## Troubleshooting — cenários reais de produção

### Cenário 1: PVC preso em Pending

```bash
kubectl get pvc meu-pvc
# STATUS: Pending   (nunca vai para Bound)

kubectl describe pvc meu-pvc | grep -A10 Events
# Possíveis causas:

# 1. WaitForFirstConsumer — normal para local-path, vai resolver quando Pod for criado
# "waiting for first consumer to be created before binding"

# 2. Nenhum PV disponível com as especificações pedidas
# "no persistent volumes available for this claim and no storage class is set"
# → Verificar se a storageClassName existe
kubectl get sc

# 3. Provisioner não está rodando (DaemonSet/Deployment do CSI driver)
# "storageclass.storage.k8s.io "nfs-homelab" not found"
kubectl get pods -n kube-system | grep -i nfs
kubectl get pods -n kube-system | grep -i csi

# 4. PV estático incompatível (access mode ou tamanho não casa)
kubectl get pv | grep Available
```

### Cenário 2: Pod preso em Pending por PVC

```bash
kubectl describe pod <nome> | grep -A5 Events
# Warning  FailedScheduling  ... persistentvolumeclaim "meu-pvc" not found
# → PVC não existe no namespace correto

# Warning  FailedScheduling  ... pod has unbound immediate PersistentVolumeClaims
# → PVC existe mas está Pending (ver cenário 1)

# Para local-path (WaitForFirstConsumer):
# O Pod ser criado é o que desencadeia o PV — verificar se o Pod está sendo agendado
kubectl describe pod <nome> | grep Node
```

### Cenário 3: PV em Released — recuperar dados

```bash
# PVC foi deletado com reclaimPolicy: Retain
# PV ficou em Released — dados intactos, mas não pode ser re-bound

kubectl get pv
# pv-nfs   10Gi   RWX   Retain   Released   (sem namespace/pvc)

# Para reutilizar o PV: remover a referência ao PVC antigo
kubectl edit pv pv-nfs
# Remover o campo spec.claimRef inteiro (ou setar para {})
# O PV volta para Available

# Para criar um novo PVC que se liga especificamente a esse PV:
# spec:
#   volumeName: pv-nfs   ← bind direto
#   storageClassName: ""  ← sem dynamic provisioning
```

### Cenário 4: Dados de local-path sumiram após node failure

```bash
# local-path armazena em /opt/local-path-provisioner/ no node específico
# Se o node morreu, os dados foram perdidos

# Verificar em qual node o PV estava
kubectl describe pv <nome> | grep -E "Node Affinity|Path"
# Node Affinity: kubernetes.io/hostname=k8s-worker1
# Path: /opt/local-path-provisioner/pvc-xxx

# O Pod vai ficar Pending porque o node não existe mais
kubectl describe pod <nome> | grep FailedScheduling
# "node(s) didn't match Pod's node affinity"

# Solução: se o node voltou, o Pod será agendado de volta
# Se o node morreu permanentemente: dados perdidos, recriar PVC em outro node
```

### Cenário 5: PVC não consegue aumentar de tamanho

```bash
kubectl patch pvc meu-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
# Error: persistentvolumeclaims "meu-pvc" is forbidden:
# only dynamically provisioned pvcas can be resized

# Verificar se a StorageClass suporta expansão
kubectl describe sc local-path | grep AllowVolume
# AllowVolumeExpansion: false  → não suporta resize

# Para local-path: não é possível fazer resize
# Solução: criar novo PVC com tamanho maior, migrar dados, recriar o Pod
```

---

## Nível avançado — edge cases e CKA/Staff

### Como o binding funciona internamente

O PersistentVolume Controller (dentro do `kube-controller-manager`) executa dois loops:

1. **Sync Claims:** para cada PVC Unbound, procura PVs disponíveis compatíveis (access mode, storage, storageClass). Se encontrar → seta `spec.claimRef` no PV e `spec.volumeName` no PVC → ambos vão para Bound.

2. **Sync Volumes:** para cada PV Released com reclaimPolicy `Delete` → deleta o PV (e o storage subjacente via provisioner).

O binding é **first-fit, not best-fit**: o Controller liga o PVC ao primeiro PV compatível encontrado, mesmo que haja um PV mais adequado. Isso pode causar desperdício de storage se você tem PVs de tamanhos variados.

### StorageClass com parâmetros específicos

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-custom
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.3.11
  share: /mnt/nfs-data/k8s/custom
  subDir: "${pvc.metadata.namespace}/${pvc.metadata.name}"  # subdir por PVC
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
- hard
- nfsvers=4.1
- timeo=600
```

### StatefulSet e volumeClaimTemplates

StatefulSets criam PVCs individuais por réplica automaticamente via `volumeClaimTemplates` — cada Pod tem seu próprio PVC dedicado, com nome previsível (`<pvc-name>-<pod-name>`).

```yaml
kind: StatefulSet
spec:
  volumeClaimTemplates:
  - metadata:
      name: dados
    spec:
      storageClassName: nfs-homelab
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 10Gi
  template:
    spec:
      containers:
      - name: db
        volumeMounts:
        - name: dados
          mountPath: /var/lib/postgresql/data
```

O PVC `dados-postgres-0` é criado para o Pod `postgres-0`, `dados-postgres-1` para `postgres-1`, etc. Quando o Pod reinicia, monta o mesmo PVC — garantia de identidade + storage estável.

> [!warning] StatefulSet PVCs não são deletados com o StatefulSet
> Deletar um StatefulSet **não deleta os PVCs** criados por `volumeClaimTemplates`. Isso é proposital (proteção de dados) mas pode deixar PVCs órfãos que consomem storage. Deletar explicitamente com `kubectl delete pvc`.

### Volume Snapshot — backup de PVCs

```yaml
# Requer VolumeSnapshotClass configurada
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: backup-postgres
spec:
  volumeSnapshotClassName: csi-nfs-snapclass
  source:
    persistentVolumeClaimName: postgres-pvc
```

### projected volume — combinar múltiplas fontes

```yaml
volumes:
- name: combined
  projected:
    sources:
    - secret:
        name: db-secret
    - configMap:
        name: app-config
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600
```

Um único mountPath recebe conteúdo de múltiplas fontes — útil para não ter vários `volumeMounts`.

### Checklist de storage em produção

```bash
# 1. Verificar StorageClasses disponíveis e a default
kubectl get sc

# 2. Verificar PVs disponíveis e seus estados
kubectl get pv

# 3. Verificar PVCs em todos os namespaces
kubectl get pvc -A | grep -v Bound   # PVCs problemáticos

# 4. Para cada PVC Pending: investigar causa
kubectl describe pvc <nome> | tail -20

# 5. Verificar uso de disco nos NFS exports
# (no servidor NFS: df -h /mnt/nfs-data/k8s)

# 6. Verificar local-path provisioner
kubectl get pods -n kube-system | grep local-path

# 7. Verificar CSI drivers
kubectl get pods -n kube-system | grep csi
```

---

## Referências

- **Kubernetes docs** — [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- **Kubernetes docs** — [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- **Kubernetes docs** — [Dynamic Volume Provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- **NFS CSI Driver** — github.com/kubernetes-csi/csi-driver-nfs
- **local-path-provisioner** — github.com/rancher/local-path-provisioner
- **Linuxtips DK8S** — `/mnt/c/Users/felip/Documents/Kubernets/DK8S`
- `kubectl explain pvc.spec` / `kubectl explain pv.spec` — referência inline da API
