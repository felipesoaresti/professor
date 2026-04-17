---
tags:
  - kubernetes
  - teoria
area: kubernetes
tipo: conteudo
next:
  - "[[configmaps]]"
  - "[[11-Exercicios/kubernetes-teoria-inicial]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Kubernetes — Teoria Inicial

> Referência: DK8S Day1–Day2 (LINUXtips) | kubernetes.io/docs
> Exercícios: [[11-Exercicios/kubernetes-teoria-inicial]] | Próximo: [[configmaps]] | Trilha: [[00-Trilha/kubernetes]]

---

## 1. O que é Kubernetes e por que existe

Antes do Kubernetes, aplicações rodavam em servidores físicos ou VMs. O problema: escalar manualmente, gerenciar falhas, distribuir carga e garantir disponibilidade era um trabalho operacional imenso e propenso a erro humano.

O **Docker** resolveu o empacotamento — "funciona na minha máquina" virou "roda em container". Mas e quando você tem dezenas de containers em produção? Quem reinicia o que caiu? Quem distribui entre servidores? Quem atualiza sem derrubar o serviço?

**Kubernetes (K8s)** é um orquestrador de containers. Ele automatiza:
- Deploy e rollout de aplicações
- Escalonamento horizontal (mais/menos réplicas)
- Auto-healing (reinicia containers que falham)
- Distribuição de carga
- Gerenciamento de configurações e secrets
- Rede entre serviços
- Armazenamento persistente

> Foi criado pelo Google (baseado no Borg interno) e doado para a CNCF em 2014.

---

## 2. Arquitetura

O cluster K8s é dividido em dois planos:

```
┌─────────────────────────────────────────────────────────┐
│                    CONTROL PLANE                        │
│  ┌──────────────┐  ┌──────────┐  ┌───────────────────┐ │
│  │  API Server  │  │   etcd   │  │  Scheduler        │ │
│  │ (kube-apiserver)│ (banco KV)│ │ (decide onde rodar)│ │
│  └──────────────┘  └──────────┘  └───────────────────┘ │
│  ┌─────────────────────────────┐                        │
│  │  Controller Manager         │                        │
│  │ (garante estado desejado)   │                        │
│  └─────────────────────────────┘                        │
└─────────────────────────────────────────────────────────┘
           │              │              │
    ┌──────┴──────┐ ┌─────┴─────┐ ┌─────┴─────┐
    │   Worker 1  │ │  Worker 2 │ │  Worker 3 │
    │  ┌────────┐ │ │ ┌───────┐ │ │ ┌───────┐ │
    │  │kubelet │ │ │ │kubelet│ │ │ │kubelet│ │
    │  │kube-   │ │ │ │kube-  │ │ │ │kube-  │ │
    │  │proxy   │ │ │ │proxy  │ │ │ │proxy  │ │
    │  └────────┘ │ │ └───────┘ │ │ └───────┘ │
    └─────────────┘ └───────────┘ └───────────┘
```

### Componentes do Control Plane

| Componente | Função |
|---|---|
| **kube-apiserver** | Ponto central de comunicação. Todo comando kubectl passa por aqui. Expõe a API REST do K8s |
| **etcd** | Banco de dados chave-valor distribuído. Armazena TODO o estado do cluster |
| **kube-scheduler** | Decide em qual node um Pod novo vai rodar (analisa recursos, afinidades, taints) |
| **kube-controller-manager** | Conjunto de controllers que garantem o estado desejado (ReplicaSet controller, Node controller, etc.) |

### Componentes dos Worker Nodes

| Componente | Função |
|---|---|
| **kubelet** | Agente que roda em cada node. Garante que os containers do Pod estão rodando |
| **kube-proxy** | Mantém regras de rede (iptables/ipvs) para comunicação entre serviços |
| **Container Runtime** | Executa os containers (containerd, CRI-O) |

### No seu HomeLab

```
k8s-master (192.168.3.30)  → control plane completo
k8s-worker1 (192.168.3.31) → kubelet + kube-proxy + containerd
k8s-worker2 (192.168.3.32) → kubelet + kube-proxy + containerd
```

---

## 3. Objetos Fundamentais

### Pod

A menor unidade do K8s. Um Pod encapsula um ou mais containers que compartilham:
- Rede (mesmo IP, mesma pilha de portas)
- Storage (volumes montados em comum)
- Ciclo de vida

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: meu-pod
  namespace: default
spec:
  containers:
  - name: app
    image: nginx:1.27
    ports:
    - containerPort: 80
```

> Pods são efêmeros. Se morrem, não voltam sozinhos. Para isso existem os controllers.

### Namespace

Particionamento lógico do cluster. Isola recursos por projeto/equipe/ambiente.

```bash
# Namespaces no seu cluster
sudo kubectl get namespaces
# giropops, controle-gastos, databases, promobot, cert-manager...
```

### Deployment

Controller que gerencia ReplicaSets e garante N réplicas de um Pod rodando. Permite rollout e rollback.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minha-app
  namespace: estudo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: minha-app
  template:
    metadata:
      labels:
        app: minha-app
    spec:
      containers:
      - name: app
        image: nginx:1.27
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

### ReplicaSet

Garante que N cópias de um Pod estão rodando. Normalmente você não cria ReplicaSets diretamente — o Deployment os gerencia.

### Service

Abstração de rede que expõe Pods. Como os Pods têm IPs efêmeros, o Service provê um IP/DNS estável.

| Tipo | Uso |
|---|---|
| **ClusterIP** | Acesso interno ao cluster (padrão) |
| **NodePort** | Expõe em uma porta de cada node (30000–32767) |
| **LoadBalancer** | Provisionamento externo (MetalLB no seu caso) |
| **ExternalName** | Alias DNS para serviço externo |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minha-app-svc
spec:
  selector:
    app: minha-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### Labels e Selectors

Labels são pares chave/valor em qualquer objeto K8s. Selectors filtram objetos por labels — é assim que Services encontram Pods, e Deployments gerenciam ReplicaSets.

```bash
# Ver pods com label app=giropops-senhas
sudo kubectl get pods -n giropops -l app=giropops-senhas
```

---

## 4. Comandos Essenciais

### Navegação e inspeção

```bash
# Listar recursos
sudo kubectl get pods -A                          # todos os pods de todos os namespaces
sudo kubectl get pods -n giropops -o wide         # wide: mostra IPs e nodes
sudo kubectl get all -n estudo                    # todos os recursos de um namespace
sudo kubectl get nodes -o wide                    # nodes com IPs

# Detalhes de um objeto
sudo kubectl describe pod <nome> -n <namespace>   # eventos, status, volumes, probes
sudo kubectl describe node k8s-worker1            # recursos, taints, pods alocados

# Ver YAML do objeto em execução
sudo kubectl get deployment giropops-senhas -n giropops -o yaml

# Logs
sudo kubectl logs <pod> -n <namespace>
sudo kubectl logs <pod> -n <namespace> -c <container>   # multi-container
sudo kubectl logs <pod> -n <namespace> --previous        # container anterior (crash)
sudo kubectl logs <pod> -n <namespace> -f                # follow (streaming)
```

### Criação e gerenciamento

```bash
# Aplicar manifesto
sudo kubectl apply -f meu-deployment.yaml

# Criar namespace
sudo kubectl create namespace estudo

# Escalar
sudo kubectl scale deployment minha-app -n estudo --replicas=5

# Rollout
sudo kubectl rollout status deployment/minha-app -n estudo
sudo kubectl rollout history deployment/minha-app -n estudo
sudo kubectl rollout undo deployment/minha-app -n estudo

# Deletar
sudo kubectl delete pod <nome> -n <namespace>
sudo kubectl delete -f meu-deployment.yaml
```

### Debug

```bash
# Shell dentro de um pod
sudo kubectl exec -it <pod> -n <namespace> -- /bin/bash

# Pod temporário para debug de rede
sudo kubectl run debug --image=nicolaka/netshoot -it --rm -- bash

# Port-forward para testar localmente
sudo kubectl port-forward svc/minha-app-svc 8080:80 -n estudo

# Eventos do cluster (ótimo para troubleshoot)
sudo kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### Geração de YAMLs

```bash
# Gerar YAML sem aplicar (--dry-run=client)
sudo kubectl create deployment teste --image=nginx --dry-run=client -o yaml > deployment.yaml

# Gerar YAML de um service
sudo kubectl expose deployment teste --port=80 --dry-run=client -o yaml
```

---

## 5. Como o K8s garante o estado desejado

O modelo declarativo do K8s: você descreve **o que quer** (não como fazer). Os controllers ficam em loop contínuo comparando o estado atual com o estado desejado e agindo para convergir.

```
Estado Desejado:  replicas: 3
Estado Atual:     replicas: 2 (1 pod morreu)
Ação do Controller: cria 1 Pod novo
```

Esse loop é chamado de **reconciliation loop** ou **control loop**.

---

## 6. Hierarquia de recursos

```
Cluster
└── Namespace
    ├── Deployment
    │   └── ReplicaSet
    │       └── Pod
    │           └── Container(s)
    ├── Service
    ├── ConfigMap
    ├── Secret
    └── PersistentVolumeClaim
```

---

## 7. Boas Práticas

- Sempre definir `requests` e `limits` de CPU/memória nos containers
- Usar `namespaces` para separar workloads (não jogar tudo no `default`)
- Labels consistentes: `app`, `version`, `component`, `env`
- Nunca editar Pods diretamente — edite o Deployment
- `kubectl apply` em vez de `kubectl create` (idempotente)
- Manter YAMLs em git (GitOps)

---

## 8. Nível Avançado / CKA

### Taints e Tolerations

Taints "repelem" Pods de nodes. Tolerations permitem que Pods específicos ignorem a repulsão.

```bash
# Adicionar taint (node só aceita workloads específicas)
kubectl taint nodes k8s-worker1 dedicated=gpu:NoSchedule

# Remover taint
kubectl taint nodes k8s-worker1 dedicated=gpu:NoSchedule-
```

No seu cluster, o control-plane tem taint por padrão:
```
node-role.kubernetes.io/control-plane:NoSchedule
```
Por isso Pods não são agendados no k8s-master.

### Node Affinity e Anti-Affinity

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
```

### Resource Quotas e LimitRanges

Controla quanto recursos um namespace pode consumir:

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

### Troubleshooting CKA

```bash
# Pod em CrashLoopBackOff
kubectl logs <pod> --previous          # ver logs antes do crash
kubectl describe pod <pod>             # ver eventos e motivo

# Pod Pending
kubectl describe pod <pod>             # ver se é falta de recursos ou taint
kubectl get events --sort-by='.lastTimestamp'

# Verificar componentes do control plane
kubectl get componentstatuses          # deprecated mas útil
kubectl get pods -n kube-system        # ver se etcd, apiserver estão OK

# Ver uso de recursos nos nodes
kubectl top nodes
kubectl top pods -A
```

---

## 9. Referências

- [Kubernetes Docs — Concepts](https://kubernetes.io/docs/concepts/)
- [Kubernetes Docs — Components](https://kubernetes.io/docs/concepts/overview/components/)
- [DK8S Day1 — LINUXtips](https://github.com/badtuxx/DescomplicandoKubernetes)
- [Badtuxx GitHub](https://github.com/badtuxx)
- [CNCF Landscape](https://landscape.cncf.io/)
