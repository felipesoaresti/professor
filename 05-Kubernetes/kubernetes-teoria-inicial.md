---
tags:
  - kubernetes
  - teoria
  - arquitetura
  - fundamentos
area: kubernetes
tipo: conteudo
next:
  - "[[05-Kubernetes/configmaps]]"
  - "[[11-Exercicios/kubernetes-teoria-inicial]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Kubernetes — Teoria Inicial

> Exercícios: [[11-Exercicios/kubernetes-teoria-inicial]] | Próximo: [[05-Kubernetes/configmaps]] | Trilha: [[00-Trilha/kubernetes]]

---

## O que é e por que existe

Antes do Kubernetes, aplicações rodavam em servidores físicos ou VMs. Escalar, recuperar de falhas, distribuir carga e atualizar sem downtime era trabalho operacional intenso e propenso a erro humano.

O Docker resolveu o empacotamento — "funciona na minha máquina" virou "roda em container". Mas com dezenas de containers em produção surgem novas perguntas: quem reinicia o que caiu? Quem distribui entre servidores? Quem atualiza sem derrubar o serviço?

**Kubernetes (K8s)** é um orquestrador de containers. Ele automatiza:
- Deploy e rollout de aplicações
- Escalonamento horizontal (mais/menos réplicas)
- Auto-healing (reinicia containers que falham)
- Balanceamento de carga interno
- Gerenciamento de configurações ([[05-Kubernetes/configmaps]]) e credenciais ([[05-Kubernetes/secrets]])
- Rede entre serviços ([[05-Kubernetes/services]])
- Armazenamento persistente ([[05-Kubernetes/volumes-storage]])

> Criado pelo Google (baseado no sistema interno Borg), doado à CNCF em 2014. Hoje é o padrão de fato para orquestração de containers.

---

## Como funciona internamente

### Arquitetura do cluster

O cluster K8s é dividido em **Control Plane** (cérebro) e **Worker Nodes** (músculo):

```
┌──────────────────────────────────────────────────────────┐
│                      CONTROL PLANE                       │
│                                                          │
│  ┌────────────────┐  ┌──────────┐  ┌─────────────────┐  │
│  │  kube-apiserver│  │   etcd   │  │  kube-scheduler │  │
│  │  (ponto central│  │ (estado  │  │ (decide onde    │  │
│  │   de tudo)     │  │  do      │  │  rodar cada Pod)│  │
│  └────────────────┘  │ cluster) │  └─────────────────┘  │
│                      └──────────┘                        │
│  ┌──────────────────────────────────────────────────┐    │
│  │  kube-controller-manager                         │    │
│  │  (ReplicaSet controller, Node controller, ...)   │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
              │              │              │
      ┌───────┴──────┐ ┌─────┴──────┐ ┌───┴────────┐
      │   Worker 1   │ │  Worker 2  │ │  Worker N  │
      │  ┌─────────┐ │ │ ┌────────┐ │ │ ┌────────┐ │
      │  │ kubelet │ │ │ │kubelet │ │ │ │kubelet │ │
      │  │kube-    │ │ │ │kube-   │ │ │ │kube-   │ │
      │  │proxy    │ │ │ │proxy   │ │ │ │proxy   │ │
      │  │containerd│ │ │ │contd  │ │ │ │contd   │ │
      │  └─────────┘ │ │ └────────┘ │ │ └────────┘ │
      └──────────────┘ └────────────┘ └────────────┘
```

### Componentes do Control Plane

| Componente                  | Função                                                                                                                   |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **kube-apiserver**          | Ponto central de comunicação. Todo `kubectl` passa aqui. Valida objetos, persiste no etcd, serve a API REST              |
| **etcd**                    | Banco chave-valor distribuído com Raft consensus. Armazena **todo** o estado do cluster — única fonte de verdade         |
| **kube-scheduler**          | Decide em qual node um Pod novo vai rodar. Analisa recursos disponíveis, afinidades, taints e restrições                 |
| **kube-controller-manager** | Conjunto de controllers rodando em loop. Cada controller gerencia um tipo de recurso (ReplicaSet, Node, Endpoints, etc.) |

### Componentes dos Worker Nodes

| Componente | Função |
|---|---|
| **kubelet** | Agente em cada node. Recebe PodSpecs do apiserver e instrui o container runtime a criar/destruir containers |
| **kube-proxy** | Mantém regras iptables/IPVS para implementar Services (ver [[05-Kubernetes/services]]) |
| **container runtime** | Executa os containers (containerd no seu cluster). Implementa a CRI (Container Runtime Interface) |

### O fluxo completo: do `kubectl apply` ao container rodando

```
kubectl apply -f deployment.yaml
       │
       ▼
kube-apiserver  ── valida o YAML, autentica, autoriza (RBAC)
       │
       ▼
  etcd  ── persiste o objeto Deployment
       │
       ▼
kube-controller-manager  ── ReplicaSet controller detecta novo Deployment
       │                    cria 3 ReplicaSets/Pods no etcd
       ▼
kube-scheduler  ── detecta Pods sem nodeName
       │           analisa recursos, taints, afinidades
       │           seta nodeName em cada Pod
       ▼
kubelet (no node escolhido)  ── detecta Pods com seu nodeName
       │                        chama containerd para criar containers
       ▼
containerd  ── faz pull da imagem, cria o container
       │
       ▼
kubelet  ── atualiza status do Pod no apiserver
```

Cada seta representa uma **watch**: o componente não "pergunta" periodicamente, ele assiste o apiserver via conexão long-lived e reage a mudanças.

### O reconciliation loop (control loop)

O modelo declarativo do K8s: você descreve **o que quer** (não como fazer). Cada controller fica em loop contínuo:

```
loop:
  estadoAtual  = observar(cluster)
  estadoDesejado = ler(etcd)
  diff = estadoDesejado - estadoAtual
  if diff != vazio:
      agir(diff)
```

Exemplo:
```
Estado Desejado:  replicas: 3
Estado Atual:     replicas: 2 (1 pod morreu)
Diff:             +1 Pod
Ação:             criar Pod novo no node com mais recurso disponível
```

Esse loop garante **self-healing** — o cluster tende sempre ao estado desejado, mesmo com falhas de hardware, rede ou software.

### Seu HomeLab

```
k8s-master  (192.168.3.30)  → control plane completo (apiserver, etcd, scheduler, controller-manager)
k8s-worker1 (192.168.3.31)  → kubelet + kube-proxy + containerd
k8s-worker2 (192.168.3.32)  → kubelet + kube-proxy + containerd
```

O master tem taint `node-role.kubernetes.io/control-plane:NoSchedule` — Pods de usuário não são agendados nele por padrão.

---

## Na prática — comandos e exemplos reais

### Objetos fundamentais

**Pod** — menor unidade. Um ou mais containers que compartilham rede e volumes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: meu-pod
  namespace: estudo
  labels:
    app: meu-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"
```

> Pods são efêmeros. Se morrem, não voltam sozinhos. Para isso existem os controllers.

### Multi-container Pods

Um Pod pode ter mais de um container. Todos os containers do Pod compartilham:
- **Rede** — mesmo IP, mesma pilha de portas (um container pode chamar outro via `localhost`)
- **Volumes** — mesmos volumes montados (podem ler e escrever o mesmo diretório)
- **Ciclo de vida** — sobem e morrem juntos

Existem três padrões de multi-container:

#### Sidecar

Roda em paralelo com o container principal. Augmenta ou enriquece o comportamento da app sem modificar sua imagem.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-com-sidecar
  namespace: estudo
spec:
  containers:
  - name: app                         # container principal
    image: nginx:1.27
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx

  - name: log-shipper                 # sidecar: lê os logs e envia para agregador
    image: fluent/fluent-bit:3.0
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx       # mesmo volume — lê os logs do nginx
    env:
    - name: FLUENT_CONF
      value: /fluent-bit/etc/fluent-bit.conf

  volumes:
  - name: logs
    emptyDir: {}
```

Casos de uso reais: proxy service mesh (Envoy/Istio), exporters de métricas, agentes de log (Fluent Bit), agentes de segurança.

#### Init Containers

Rodam **antes** dos containers principais, em sequência. Cada init container deve completar com sucesso antes do próximo iniciar. O container principal só sobe quando todos os init containers terminam.

```yaml
k app
```

> [!info] Init vs Sidecar
> Init containers **terminam** — sua função é preparar o ambiente. Sidecar containers **ficam rodando** junto com o container principal. Se um init container falha, o Pod entra em `Init:CrashLoopBackOff`.

```bash
# Ver o estado dos init containers
kubectl describe pod app-com-init -n estudo
# Init Containers:
#   wait-for-db: State: Terminated, Exit Code: 0
#   run-migrations: State: Running

# Logs de um init container específico
kubectl logs app-com-init -n estudo -c wait-for-db
```

#### Ephemeral Containers (debug)

Adicionados dinamicamente a um Pod **em execução**, sem reiniciar o Pod. Útil para debugar containers sem shell ou sem ferramentas de diagnóstico.

```bash
# Adicionar container de debug ao Pod em execução
kubectl debug -it <pod> -n estudo \
  --image=nicolaka/netshoot \
  --target=app              # compartilha o namespace de processo do container "app"

# Listar ephemeral containers ativos
kubectl get pod <pod> -n estudo -o jsonpath='{.spec.ephemeralContainers}'
```

> [!warning] Ephemeral containers são imutáveis após criados
> Não é possível removê-los do Pod. Eles somem apenas quando o Pod é reiniciado/recriado.

**Deployment** — gerencia ReplicaSets, garante N réplicas, permite rollout/rollback.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: estudo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        version: "1.0"
    spec:
      containers:
      - name: app
        image: nginx:1.27
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
```

**Service** — IP/DNS estável para um grupo de Pods (selecionados por labels).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
  namespace: estudo
spec:
  selector:
    app: webapp          # encontra Pods com este label
  ports:
  - port: 80             # porta do Service
    targetPort: 80       # porta no container
  type: ClusterIP        # apenas acessível dentro do cluster
```

**Hierarquia de recursos:**

```
Cluster
└── Namespace
    ├── Deployment → ReplicaSet → Pod → Container(s)
    ├── Service  →  Endpoints (lista de IPs de Pods)
    ├── ConfigMap
    ├── Secret
    ├── PersistentVolumeClaim → PersistentVolume
    └── Ingress → Service
```

### Labels e Selectors — a cola do K8s

Labels são pares chave/valor em qualquer objeto. Selectors filtram por labels. Toda a "cola" do K8s funciona por labels:
- Service encontra Pods pelo selector
- Deployment gerencia ReplicaSet pelo selector
- ReplicaSet gerencia Pods pelo selector

```bash
# Ver pods por label
sudo kubectl get pods -n estudo -l app=webapp

# Ver pods com múltiplos labels
sudo kubectl get pods -n estudo -l app=webapp,env=producao

# Adicionar label a um pod existente
sudo kubectl label pod meu-pod -n estudo version=1.0

# Ver labels de todos os pods
sudo kubectl get pods -n estudo --show-labels
```

### Comandos essenciais

```bash
# Listar recursos
sudo kubectl get pods -A                         # todos os pods de todos os namespaces
sudo kubectl get pods -n estudo -o wide          # wide: IPs e nodes
sudo kubectl get all -n estudo                   # todos os recursos do namespace
sudo kubectl get nodes -o wide                   # nodes com IPs e versões

# Detalhes
sudo kubectl describe pod <nome> -n estudo       # eventos, status, volumes, probes
sudo kubectl describe node k8s-worker1           # recursos, taints, pods alocados

# YAML do objeto em execução
sudo kubectl get deployment webapp -n estudo -o yaml

# Logs
sudo kubectl logs <pod> -n estudo
sudo kubectl logs <pod> -n estudo -c <container> # multi-container
sudo kubectl logs <pod> -n estudo --previous     # container anterior (crash)
sudo kubectl logs <pod> -n estudo -f             # follow (streaming)

# Criação e gerenciamento
sudo kubectl apply -f deployment.yaml            # idempotente (preferir sobre create)
sudo kubectl create namespace estudo
sudo kubectl scale deployment webapp -n estudo --replicas=5

# Rollout
sudo kubectl rollout status deployment/webapp -n estudo
sudo kubectl rollout history deployment/webapp -n estudo
sudo kubectl rollout undo deployment/webapp -n estudo
sudo kubectl rollout undo deployment/webapp -n estudo --to-revision=2

# Debug
sudo kubectl exec -it <pod> -n estudo -- /bin/sh
sudo kubectl run debug --image=nicolaka/netshoot -it --rm -- bash
sudo kubectl port-forward svc/webapp-svc 8080:80 -n estudo

# Eventos (ótimo para troubleshoot)
sudo kubectl get events -n estudo --sort-by='.lastTimestamp'

# Gerar YAMLs sem aplicar (dry-run)
sudo kubectl create deployment teste --image=nginx:1.27 --dry-run=client -o yaml
sudo kubectl expose deployment teste --port=80 --dry-run=client -o yaml

# Referência de campos de qualquer objeto (essencial no CKA)
sudo kubectl explain pod.spec.containers
sudo kubectl explain deployment.spec.template.spec --recursive
```

---

## Casos de uso e boas práticas

**Sempre definir `requests` e `limits`** — sem `requests`, o scheduler não sabe onde alocar. Sem `limits`, um container pode consumir todos os recursos do node.

**Nunca usar `latest` como tag de imagem** — cria builds não-reproduzíveis e impede cache eficiente nos nodes.

**Namespaces por workload/equipe** — nunca jogar tudo no `default`. Permite RBAC, quotas e isolamento.

**Labels consistentes** — use o padrão recomendado pelo Kubernetes:
```yaml
labels:
  app.kubernetes.io/name: webapp
  app.kubernetes.io/version: "1.2.3"
  app.kubernetes.io/component: frontend
  app.kubernetes.io/part-of: ecommerce
  app.kubernetes.io/managed-by: helm
```

**`kubectl apply` em vez de `kubectl create`** — `apply` é idempotente: funciona tanto para criar quanto para atualizar.

**Nunca editar Pods diretamente** — edite o Deployment. Pods gerenciados por um controller são substituídos pelo controller ao serem deletados; mudanças diretas são perdidas.

**YAMLs em git** — o estado do cluster deve ser representado em código (GitOps). O que não está em git não existe.

**Containers não devem rodar como root** — use `securityContext.runAsNonRoot: true` e `runAsUser` diferente de 0.

---

## Troubleshooting — cenários reais de produção

### Pod em `Pending`

```bash
kubectl describe pod <pod> -n <ns>
```

Causas comuns nos Events:
- `Insufficient cpu/memory` → node sem recursos
- `0/2 nodes are available: 1 node(s) had untolerated taint` → taint no node sem toleration no Pod
- `no persistent volumes available for this claim` → PVC sem PV disponível

### Pod em `CrashLoopBackOff`

```bash
kubectl logs <pod> -n <ns> --previous   # logs do crash anterior
kubectl describe pod <pod> -n <ns>      # exit code nos eventos
```

Exit codes comuns: `1` = erro da aplicação, `137` = OOM killed (`SIGKILL`), `143` = graceful shutdown (`SIGTERM`).

### Pod em `CreateContainerConfigError`

```bash
kubectl get events -n <ns> --sort-by='.lastTimestamp'
```

Causa típica: ConfigMap ou Secret referenciado não existe.

### Pod em `ImagePullBackOff`

```bash
kubectl describe pod <pod> -n <ns>
# Events: Failed to pull image "nginx:nao-existe": ...
```

Causa: imagem inexistente, tag errada, ou registry privado sem imagePullSecret.

### Service sem Endpoints (aplicação inacessível)

```bash
kubectl get endpoints <svc> -n <ns>
# <none> = nenhum Pod com o label certo

kubectl get pods -n <ns> --show-labels
# Comparar labels dos Pods com o selector do Service
```

Causa quase sempre: mismatch de labels entre Service selector e Pod labels.

### Metodologia sistemática de diagnóstico

```
1. kubectl get pods → qual é o estado?
2. kubectl describe pod → qual é o motivo?
3. kubectl logs → o que a aplicação disse?
4. kubectl get events → o que o K8s fez?
5. kubectl get endpoints → o Service enxerga os Pods?
```

---

## Nível avançado — edge cases e cenários CKA/Staff

### Taints, Tolerations e por que o master não recebe Pods

```bash
# Ver taints do master
sudo kubectl describe node k8s-master | grep Taints
# Taints: node-role.kubernetes.io/control-plane:NoSchedule

# Adicionar taint a um worker
sudo kubectl taint nodes k8s-worker1 dedicated=gpu:NoSchedule

# Remover taint
sudo kubectl taint nodes k8s-worker1 dedicated=gpu:NoSchedule-

# Toleration no Pod para aceitar o taint
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "gpu"
  effect: "NoSchedule"
```

DaemonSets do sistema (como Calico, kube-proxy) têm tolerations para `NoSchedule` — por isso rodam até no master. Ver [[05-Kubernetes/daemonset]].

### Node Affinity

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-worker1
          - k8s-worker2
```

`requiredDuring...` = hard (Pod não é alocado se não atender). `preferredDuring...` = soft (tenta atender, mas não bloqueia).

### ResourceQuota — limitar namespace inteiro

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: estudo-quota
  namespace: estudo
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
```

```bash
sudo kubectl describe resourcequota estudo-quota -n estudo
# Mostra usado vs limite
```

### Como inspecionar qualquer campo via `kubectl explain`

No CKA você não tem internet, mas tem `kubectl explain`:

```bash
# Estrutura completa de um recurso
kubectl explain pod --recursive
kubectl explain deployment.spec.strategy

# Campo específico com descrição
kubectl explain pod.spec.containers.resources
kubectl explain pod.spec.containers.livenessProbe
```

### Ver componentes do control plane

```bash
# Pods dos componentes (instalados como static pods)
sudo kubectl get pods -n kube-system
# kube-apiserver-k8s-master, etcd-k8s-master, kube-scheduler-k8s-master, etc.

# Static pods ficam em:
ls /etc/kubernetes/manifests/
# etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml

# Uso de recursos nos nodes
sudo kubectl top nodes
sudo kubectl top pods -A
```

### `kubectl diff` — ver o que vai mudar antes de aplicar

```bash
sudo kubectl diff -f deployment.yaml
# Mostra diff entre o que está no cluster e o que seria aplicado
```

---

## Referências

- [Kubernetes Docs — Concepts](https://kubernetes.io/docs/concepts/)
- [Kubernetes Docs — Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Kubernetes API Reference](https://kubernetes.io/docs/reference/kubernetes-api/)
- [DK8S — Descomplicando Kubernetes (LINUXtips)](https://github.com/badtuxx/DescomplicandoKubernetes)
- [CNCF Landscape](https://landscape.cncf.io/)
- Próximos assuntos: [[05-Kubernetes/configmaps]] | [[05-Kubernetes/secrets]] | [[05-Kubernetes/services]] | [[05-Kubernetes/deployment-strategies]]
