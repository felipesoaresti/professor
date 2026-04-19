---
tags:
  - kubernetes
  - daemonset
  - workloads
  - observabilidade
area: kubernetes
tipo: conteudo
prerequisites:
  - "[[05-Kubernetes/kubernetes-teoria-inicial]]"
  - "[[05-Kubernetes/probes]]"
next:
  - "[[11-Exercicios/daemonset]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# DaemonSet

> Conteúdo: [[05-Kubernetes/daemonset]] | Exercícios: [[11-Exercicios/daemonset]] | Trilha: [[00-Trilha/kubernetes]]

O DaemonSet garante que exatamente **um Pod rode em cada node** do cluster — ou em um subconjunto de nodes definido por labels. É o mecanismo correto para qualquer workload que precisa existir em toda máquina: agentes de monitoramento, coletores de log, plugins de rede, agentes de segurança.

---

## O que é e por que existe

Um Deployment com `replicas: 3` em um cluster de 10 nodes distribui 3 Pods em 3 nodes. Se você adicionar um 11º node, nada muda — o Deployment não sabe. Se você precisa que **todo node tenha o agente do CheckMK**, um Deployment não serve.

O DaemonSet resolve isso: quando um novo node entra no cluster, o DaemonSet Controller automaticamente agenda um Pod nele. Quando o node é removido, o Pod é deletado. Não há campo `replicas` — o número de Pods é sempre igual ao número de nodes elegíveis.

**Casos de uso canônicos:**
- Agentes de monitoramento: `node-exporter`, `CheckMK agent`, `Datadog agent`
- Coletores de log: `Fluentd`, `Filebeat`, `Promtail`
- Plugins de rede: `Calico node`, `Flannel`, `WeaveNet` (você já usa Calico no homelab)
- Armazenamento: `ceph-osd`, drivers CSI
- Segurança: `Falco`, agentes de compliance

---

## Como funciona internamente

### DaemonSet Controller

O DaemonSet Controller fica no loop de reconciliação do `kube-controller-manager`. Para cada node do cluster, ele verifica:

1. O node satisfaz o `nodeSelector` / `nodeAffinity` do DaemonSet?
2. O Pod tolera os taints do node?
3. Já existe um Pod deste DaemonSet neste node?

Se (1) e (2) são verdadeiros e (3) é falso → cria o Pod.  
Se (1) ou (2) são falsos e existe um Pod → deleta o Pod.

### Como o Pod é agendado no node

O DaemonSet Pod **não passa pelo kube-scheduler** da mesma forma que Pods normais. O Controller seta `spec.nodeName` diretamente no Pod antes de criá-lo — o node é pré-definido, não escolhido pelo scheduler. Isso garante que o Pod vá para o node correto mesmo se o scheduler estiver sobrecarregado ou se o node estiver marcado como `unschedulable` (como durante um drain).

> [!info] DaemonSet e kubectl drain
> Por padrão, `kubectl drain` remove todos os Pods de um node — exceto DaemonSet Pods. Para remover também os DaemonSet Pods durante o drain: `kubectl drain <node> --ignore-daemonsets`. Os Pods são recriados quando o node volta.

### Diferenças do Deployment no spec

```yaml
# Deployment tem:
spec:
  replicas: 3          # ← não existe no DaemonSet
  selector: ...
  template: ...

# DaemonSet tem:
spec:
  selector: ...        # obrigatório — deve casar com template.labels
  template: ...        # igual ao Deployment
  updateStrategy: ...  # estratégia de update (diferente do Deployment)
```

### Update strategies do DaemonSet

Diferente do Deployment, o DaemonSet tem suas próprias strategies:

**`RollingUpdate` (padrão):** atualiza os Pods node por node, controlado por `maxUnavailable`. Não existe `maxSurge` — em um DaemonSet não faz sentido ter dois Pods no mesmo node.

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1      # quantos nodes podem ter o DaemonSet Pod indisponível (padrão: 1)
    maxSurge: 0            # sempre 0 para DaemonSets (não configurável de outra forma)
```

**`OnDelete`:** o Pod **não é atualizado automaticamente**. Para atualizar, você precisa deletar manualmente o Pod no node que quer atualizar. O Controller cria um novo Pod com o template atualizado.

```yaml
updateStrategy:
  type: OnDelete
```

> [!tip] Quando usar OnDelete
> Útil para atualizações controladas de componentes críticos de infraestrutura (ex: plugin de rede), onde você quer controlar exatamente quando cada node é afetado — geralmente durante uma janela de manutenção.

---

## Na prática — YAMLs e comandos

### DaemonSet básico — agente de monitoramento simulado

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
  namespace: monitoring
  labels:
    app: node-agent
spec:
  selector:
    matchLabels:
      app: node-agent
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane  # roda também no master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: agent
        image: busybox:1.36
        command: ["/bin/sh", "-c"]
        args:
        - |
          while true; do
            echo "$(date): coletando métricas do node $(NODE_NAME)"
            sleep 30
          done
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName   # injeta o nome do node onde o Pod está
        resources:
          requests:
            cpu: 50m
            memory: 32Mi
          limits:
            cpu: 100m
            memory: 64Mi
```

### Rodando no control-plane (master)

Por padrão, o control-plane tem um taint que impede Pods normais:
```
node-role.kubernetes.io/control-plane:NoSchedule
```

Para que o DaemonSet rode também no master, adicione a toleration:

```yaml
tolerations:
- key: node-role.kubernetes.io/control-plane
  operator: Exists
  effect: NoSchedule
```

Verifique os taints do seu cluster:
```bash
kubectl describe node k8s-master | grep Taint
# Taints: node-role.kubernetes.io/control-plane:NoSchedule
```

### DaemonSet apenas em subset de nodes (nodeSelector)

```yaml
spec:
  template:
    spec:
      nodeSelector:
        node-type: storage    # só nodes com esse label
```

```bash
# Adicionar label a um node
kubectl label node k8s-worker1 node-type=storage

# Remover label (DaemonSet Pod será deletado do node)
kubectl label node k8s-worker1 node-type-
```

### DaemonSet com nodeAffinity (mais expressivo)

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
              - key: node-role.kubernetes.io/worker
                operator: Exists
```

### Acessando o filesystem do host (hostPath)

DaemonSets frequentemente precisam ler arquivos do node host — logs, métricas de kernel, certificados.

```yaml
containers:
- name: log-collector
  image: busybox:1.36
  volumeMounts:
  - name: varlog
    mountPath: /host/var/log
    readOnly: true
  - name: docker-sock
    mountPath: /var/run/docker.sock
volumes:
- name: varlog
  hostPath:
    path: /var/log
    type: Directory
- name: docker-sock
  hostPath:
    path: /var/run/docker.sock
    type: Socket
```

### hostNetwork — usar a rede do node diretamente

```yaml
spec:
  template:
    spec:
      hostNetwork: true    # Pod usa a stack de rede do node, não a do Pod
      hostPID: true        # acesso ao PID namespace do host
      dnsPolicy: ClusterFirstWithHostNet  # obrigatório com hostNetwork
```

> [!warning] hostNetwork tem implicações de segurança
> Com `hostNetwork: true`, o Pod pode escutar em qualquer porta do node. Use apenas quando absolutamente necessário (plugins de rede, agentes de baixo nível). Sempre combine com `SecurityContext` restritivo.

### Comandos essenciais

```bash
# Listar DaemonSets
kubectl get daemonset -A
kubectl get ds -n monitoring

# Ver detalhes — DESIRED/CURRENT/READY/UP-TO-DATE/AVAILABLE/NODE SELECTOR
kubectl describe ds node-agent -n monitoring

# Ver em quais nodes o DaemonSet está rodando
kubectl get pods -l app=node-agent -o wide

# Ver logs de um Pod específico de um node
kubectl logs -l app=node-agent --field-selector spec.nodeName=k8s-worker1

# Rollout status de um DaemonSet (igual ao Deployment)
kubectl rollout status daemonset/node-agent -n monitoring

# Rollout history
kubectl rollout history daemonset/node-agent -n monitoring

# Rollback
kubectl rollout undo daemonset/node-agent -n monitoring

# Forçar atualização de um Pod específico (strategy OnDelete)
kubectl delete pod <pod-no-node-x> -n monitoring
# O Controller recria com o template novo
```

### Injetando informações do node no container

```yaml
env:
- name: NODE_NAME
  valueFrom:
    fieldRef:
      fieldPath: spec.nodeName
- name: NODE_IP
  valueFrom:
    fieldRef:
      fieldPath: status.hostIP
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: POD_NAMESPACE
  valueFrom:
    fieldRef:
      fieldPath: metadata.namespace
```

---

## Casos de uso e boas práticas

### Calico node — o DaemonSet que você já tem no cluster

O Calico CNI já roda como DaemonSet no seu cluster:

```bash
kubectl get ds -n kube-system
# calico-node   3   3   3   3   3   kubernetes.io/os=linux
```

```bash
kubectl describe ds calico-node -n kube-system | head -40
# Observe: tolerations para control-plane, hostNetwork: true, hostPID: true,
# volumes com hostPath para /var/lib/calico, /var/run/calico
# Esse é o padrão de um DaemonSet de plugin de rede de produção
```

### Resources: seja conservador

DaemonSet Pods competem com workloads em todos os nodes. Um DaemonSet com `limits.memory: 2Gi` em um cluster de 10 nodes reserva 20Gi para o agente. Seja conservador:

```yaml
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

### PriorityClass para DaemonSets críticos

DaemonSets de infraestrutura (plugin de rede, agente de segurança) devem ter prioridade mais alta que workloads comuns para não serem evicted quando o node fica com pouca memória:

```yaml
spec:
  template:
    spec:
      priorityClassName: system-node-critical  # prioridade máxima do sistema
```

Classes disponíveis por padrão:
- `system-cluster-critical` — usado pelo kube-dns, metrics-server
- `system-node-critical` — usado pelo calico-node, kube-proxy

### ServiceAccount e RBAC para DaemonSets

DaemonSets que precisam fazer chamadas à API (listar nodes, ler secrets) precisam de ServiceAccount com permissões adequadas:

```yaml
spec:
  template:
    spec:
      serviceAccountName: node-agent
      automountServiceAccountToken: true
```

```yaml
# ClusterRole para ler informações de nodes
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-agent
rules:
- apiGroups: [""]
  resources: ["nodes", "pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-agent
subjects:
- kind: ServiceAccount
  name: node-agent
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: node-agent
  apiGroup: rbac.authorization.k8s.io
```

---

## Troubleshooting — cenários reais de produção

### Cenário 1: DaemonSet não cria Pod no master

```bash
# Sintoma: DESIRED=3 mas apenas 2 nodes têm o Pod (worker1, worker2 — sem o master)
kubectl get ds node-agent
# NAME        DESIRED   CURRENT   READY   ...
# node-agent  3         2         2       ...

# Diagnóstico: verificar taint do control-plane
kubectl describe node k8s-master | grep -i taint
# Taints: node-role.kubernetes.io/control-plane:NoSchedule

# O DaemonSet não tem a toleration para esse taint
kubectl get ds node-agent -o jsonpath='{.spec.template.spec.tolerations}'

# Solução: adicionar toleration ao DaemonSet
kubectl patch ds node-agent --type=json -p='[{
  "op": "add",
  "path": "/spec/template/spec/tolerations/-",
  "value": {
    "key": "node-role.kubernetes.io/control-plane",
    "operator": "Exists",
    "effect": "NoSchedule"
  }
}]'
```

### Cenário 2: DaemonSet não cria Pod em node novo

```bash
# Novo node adicionado ao cluster mas o DaemonSet não agendou Pod nele

# 1. Verificar se o node está Ready
kubectl get node <novo-node>

# 2. Verificar labels do node (nodeSelector do DaemonSet precisa casar)
kubectl get node <novo-node> --show-labels
kubectl describe ds node-agent | grep -A5 "Node-Selector"

# 3. Verificar taints do node
kubectl describe node <novo-node> | grep Taint
# Se o node tem taint que o DaemonSet não tolera → Pod não é criado

# 4. Verificar eventos do DaemonSet
kubectl describe ds node-agent | grep -A20 Events
```

### Cenário 3: Pod do DaemonSet em Pending

```bash
kubectl get pods -l app=node-agent -o wide
# node-agent-xxxxx   0/1   Pending   k8s-worker2

kubectl describe pod node-agent-xxxxx
# Events:
# Warning  FailedScheduling  ... Insufficient memory

# Causa: o node não tem recursos para o DaemonSet Pod
# (requests muito altos ou node com muita carga)

# Verificar recursos disponíveis no node
kubectl describe node k8s-worker2 | grep -A10 "Allocated resources"

# Solução: reduzir requests do DaemonSet ou liberar recursos no node
```

### Cenário 4: Update travado — DaemonSet rollingUpdate não progride

```bash
kubectl rollout status daemonset/node-agent
# Waiting for daemon set "node-agent" rollout to finish:
# 1 out of 3 new pods have been updated...

# Pod novo não está ficando Ready
kubectl get pods -l app=node-agent -o wide
# node-agent-new-xxx  0/1  Running  k8s-worker1  ← nunca fica 1/1

# Causa provável: readinessProbe falhando na nova versão
kubectl describe pod node-agent-new-xxx | grep -A10 Events

# Se necessário, rollback
kubectl rollout undo daemonset/node-agent
```

### Cenário 5: Investigar DaemonSet de um CNI com problema de rede

```bash
# Suspeita: calico-node Pod travado em um worker está causando problemas de rede

# Ver estado do calico-node em todos os workers
kubectl get pods -n kube-system -l k8s-app=calico-node -o wide

# Logs do calico-node no worker problemático
kubectl logs -n kube-system calico-node-xxxxx -c calico-node --tail=50

# Reiniciar o Pod problemático (DaemonSet vai recriar)
kubectl delete pod calico-node-xxxxx -n kube-system

# Monitorar
kubectl get pods -n kube-system -l k8s-app=calico-node -w
```

---

## Nível avançado — edge cases e CKA/Staff

### DaemonSet vs Deployment vs StaticPod

| | DaemonSet | Deployment | StaticPod |
|---|---|---|---|
| **Garantia** | 1 Pod por node elegível | N réplicas no cluster | 1 Pod em node específico |
| **Gerenciado por** | DaemonSet Controller | ReplicaSet Controller | kubelet diretamente |
| **Tolerations necessárias** | Sim, para nodes com taints | Sim | Não (não passa pelo scheduler) |
| **Sobrevive ao API server?** | Não | Não | Sim |
| **Casos de uso** | Agentes, plugins | Apps stateless | kube-apiserver, etcd |

StaticPods são definidos como arquivos YAML em `/etc/kubernetes/manifests/` no node. O kubelet os lê diretamente, sem API server. É assim que os componentes do control-plane (apiserver, etcd, scheduler) rodam — eles são StaticPods, não DaemonSets.

```bash
# Ver StaticPods do control-plane
ls /etc/kubernetes/manifests/
# etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml

kubectl get pods -n kube-system | grep -E "apiserver|etcd|scheduler|controller"
# Esses pods têm o sufixo do nome do node: kube-apiserver-k8s-master
```

### maxUnavailable como percentual

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 33%  # em cluster de 3 nodes = 1 node por vez
```

Com percentual, o DaemonSet atualiza N% dos nodes por vez — útil em clusters grandes para controlar o ritmo de atualização.

### `minReadySeconds` no DaemonSet

```yaml
spec:
  minReadySeconds: 30   # aguarda 30s após Pod Ready antes de avançar para o próximo node
```

Funciona exatamente como no Deployment — protege contra bugs que aparecem segundos após o container iniciar.

### DaemonSet com hostPort — expor diretamente no node

```yaml
containers:
- name: agent
  ports:
  - containerPort: 9100
    hostPort: 9100       # expõe na porta do node diretamente
    protocol: TCP
```

Com `hostPort`, o Pod é acessível via `<node-ip>:9100` sem precisar de Service. O `node-exporter` do Prometheus usa esse padrão — o Prometheus scrapa cada node diretamente no IP do node.

> [!warning] hostPort limita o scheduling
> Com `hostPort`, só pode existir um Pod usando aquela porta em cada node. O scheduler considera isso como um recurso — se a porta já estiver em uso, o Pod não é agendado.

### Comparar DaemonSet com Deployment para o mesmo workload

**Pergunta de CKA:** "Você tem um agente de coleta de logs. Por que DaemonSet e não Deployment com `replicas: 3` (um por node)?"

Resposta correta:
1. Com Deployment, se você adicionar um 4º node, ele não terá o agente automaticamente — você teria que aumentar `replicas` manualmente
2. Com Deployment, não há garantia de que cada node terá exatamente 1 Pod — o scheduler pode colocar 2 no mesmo node e 0 em outro
3. Com DaemonSet, a garantia é automática e bidirecional: node entra → Pod criado; node sai → Pod deletado

### Ler DaemonSet nos nodes com Downward API + hostPath

```yaml
# Padrão avançado: agente que reporta seu próprio node e lê métricas do host
containers:
- name: agent
  env:
  - name: MY_NODE_NAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName
  volumeMounts:
  - name: proc
    mountPath: /host/proc
    readOnly: true
  - name: sys
    mountPath: /host/sys
    readOnly: true
volumes:
- name: proc
  hostPath:
    path: /proc
- name: sys
  hostPath:
    path: /sys
```

Com acesso ao `/proc` e `/sys` do host, o agente pode coletar todas as métricas do node — exatamente o que o `node-exporter` faz.

---

## Referências

- **Kubernetes docs** — [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- **Kubernetes docs** — [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- **node-exporter** — github.com/prometheus/node_exporter (exemplo real de DaemonSet)
- **Calico source** — github.com/projectcalico/calico (DaemonSet de CNI em produção no homelab)
- **Linuxtips DK8S** — `/mnt/c/Users/felip/Documents/Kubernets/DK8S`
- `kubectl explain daemonset.spec` — referência inline da API
