---
tags:
  - exercicios
  - kubernetes
  - kubeadm
  - cni
  - taints
  - kind
tipo: exercicios
area: kubernetes
conteudo: "[[05-Kubernetes/cluster-kubeadm-cni-kind]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Exercícios — Cluster Kubernetes: kubeadm, CNI, Taints e Kind

> Conteúdo: [[05-Kubernetes/cluster-kubeadm-cni-kind]] | Trilha: [[00-Trilha/kubernetes]]

> [!warning] Restrições
> Os exercícios de kubeadm são realizados em uma VM separada ou no node `study` (192.168.3.20) — **nunca no cluster de produção** (k8s-master/worker1/worker2).
> Exercícios de Taints e Kind podem ser executados localmente (WSL2) ou no cluster de produção no namespace `estudo`.

---

## Exercício 1: Anatomia do Cluster Existente

**Contexto:** Antes de criar um cluster do zero, é essencial entender o que kubeadm construiu no cluster que você já opera. A maioria dos engenheiros usa clusters sem nunca olhar o que está por baixo.

**Missão:** Fazer um inventário completo dos artefatos que o kubeadm gerou no cluster de produção.

**Requisitos:**
- [ ] No k8s-master, listar todos os certificados e suas datas de expiração
- [ ] Identificar qual CA é raiz de quais certificados (PKI tree)
- [ ] Listar os static pods em `/etc/kubernetes/manifests/` e verificar qual versão de imagem cada um usa
- [ ] Inspecionar o ConfigMap `kubeadm-config` no namespace `kube-system` e identificar: versão do Kubernetes, podSubnet, serviceSubnet, e o controlPlaneEndpoint configurado
- [ ] Verificar quais flags extras foram passadas para o kube-apiserver (campo `extraArgs` no ConfigMap ou diretamente no static pod manifest)

**Verificação:**
```bash
# No k8s-master
sudo kubeadm certs check-expiration

sudo ls -la /etc/kubernetes/manifests/

sudo kubectl -n kube-system get cm kubeadm-config -o yaml
```

> [!tip] Dica de Investigação
> O ConfigMap `kubeadm-config` guarda a `ClusterConfiguration` original. Compare com os argumentos reais dos static pods em `/etc/kubernetes/manifests/kube-apiserver.yaml` — pode haver divergências por edições manuais posteriores.

---

## Exercício 2: Analisar o CNI em Produção (Calico)

**Contexto:** O cluster usa Calico CNI, mas a maioria das pessoas que trabalha nele nunca olhou como ele funciona por baixo. Você precisa entender o mecanismo de roteamento antes de tomar decisões de networking.

**Missão:** Inspecionar como o Calico está operando no cluster — modo de roteamento, tabelas de IPs alocados, e conectividade entre Pods em nodes diferentes.

**Requisitos:**
- [ ] Identificar em qual modo o Calico está operando: BGP puro, VXLAN, ou IPIP (inspecionar o IPPool e verificar o campo `encapsulation`)
- [ ] Listar os BGP peers que o Calico estabeleceu (nodes se peerizando entre si)
- [ ] Verificar a tabela de rotas em um dos workers — identificar qual rota corresponde à subnet de Pods do outro worker
- [ ] Criar dois Pods no namespace `estudo` — um no worker1 e um no worker2 (usando `nodeSelector`) — e fazer ping entre eles
- [ ] Usando `tcpdump` ou `ip route` em um dos nodes durante o ping, confirmar se há encapsulation (VXLAN) ou roteamento direto

**Verificação:**
```bash
# Listar IPPools do Calico
sudo kubectl get ippool -o yaml

# Listar BGP peers (requer calicoctl ou kubectl com CRDs do Calico)
sudo kubectl get bgppeer -A

# Tabela de rotas do node
ip route show | grep -v kernel
```

---

## Exercício 3: Taints no Cluster de Produção

**Contexto:** O scheduler decide onde colocar cada Pod com base em taints, tolerations e affinity. Você precisa entender o estado atual e experimentar com taints sem afetar workloads de produção.

**Missão:** Mapear os taints existentes, criar um cenário com taint customizado e observar o comportamento do scheduler.

**Requisitos:**
- [ ] Listar os taints de todos os nodes do cluster (usar `custom-columns` para output legível)
- [ ] Verificar que o control-plane tem o taint `node-role.kubernetes.io/control-plane:NoSchedule` e entender por que nenhum Pod de usuário roda nele (verificar quais Pods têm toleration para esse taint)
- [ ] Aplicar o taint `dedicated=estudo:NoSchedule` no **worker1** apenas
- [ ] Criar um Deployment com 3 réplicas no namespace `estudo` **sem** toleration — verificar que todas as réplicas vão para o worker2
- [ ] Adicionar a toleration correta ao Deployment — verificar que réplicas são distribuídas entre ambos os workers
- [ ] Remover o taint do worker1 ao finalizar

**Verificação:**
```bash
# Listar taints
sudo kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints'

# Verificar onde os Pods foram agendados
sudo kubectl get pods -n estudo -o wide
```

---

## Exercício 4: Taint NoExecute — Eviction em Tempo Real

**Contexto:** Um node está com comportamento instável (alta latência de disco, por exemplo). Você precisa esvaziar o node de forma controlada sem fazer `kubectl drain` — usando NoExecute taint com `tolerationSeconds` para dar grace period diferente por tipo de workload.

**Missão:** Demonstrar o comportamento de NoExecute com tolerationSeconds diferente para Pods distintos.

**Requisitos:**
- [ ] Criar no namespace `estudo` dois Deployments, ambos forçados para o **worker1** via `nodeSelector`:
  - `app: rapido` — com toleration `NoExecute` e `tolerationSeconds: 30`
  - `app: lento` — **sem** toleration para NoExecute (usa o default do Kubernetes: 300s)
- [ ] Após criar os Pods e confirmar que estão no worker1, aplicar o taint `manutencao=true:NoExecute` no worker1
- [ ] Observar a diferença: `rapido` é evicted em ~30s, `lento` fica por ~5 minutos (ou até ser removido manualmente)
- [ ] Verificar em que node os Pods reagendados foram para
- [ ] Remover o taint do worker1 ao finalizar

**Verificação:**
```bash
# Watch nos Pods para ver a eviction em tempo real
sudo kubectl get pods -n estudo -o wide -w

# Verificar eventos de eviction
sudo kubectl get events -n estudo --sort-by='.lastTimestamp' | grep -i evict
```

> [!info] Por que o default é 300s
> O Kubernetes adiciona automaticamente tolerations para `not-ready` e `unreachable` com 300s em todos os Pods. Isso define o tempo que um node precisa ficar offline para seus Pods serem reagendados. Pode ser configurado via `--default-not-ready-toleration-seconds` e `--default-unreachable-toleration-seconds` no controller-manager.

---

## Exercício 5: Node Dedicado com Taint + NodeAffinity

**Contexto:** O time de ML quer um node dedicado para workloads de machine learning. Apenas Pods com toleration correta podem entrar, E os Pods de ML devem **preferir** esse node (mas não travar se ele estiver indisponível).

**Missão:** Configurar um node dedicado usando a combinação correta de Taint + NodeAffinity.

**Requisitos:**
- [ ] Aplicar no **worker2**: taint `workload=ml:NoSchedule` e label `workload=ml`
- [ ] Criar um Deployment `ml-workload` com:
  - Toleration para o taint `workload=ml:NoSchedule`
  - `nodeAffinity` com `preferredDuringSchedulingIgnoredDuringExecution` para o label `workload=ml` (soft — não obrigatório)
  - 3 réplicas
- [ ] Criar um Deployment `generic-workload` **sem** toleration e verificar que vai apenas para o worker1
- [ ] Escalar `ml-workload` para 5 réplicas e verificar que réplicas extras vão para o worker1 (porque a affinity é soft)
- [ ] Explicar: qual seria o comportamento se a affinity fosse `requiredDuringSchedulingIgnoredDuringExecution`?
- [ ] Limpar taints e labels do worker2 ao finalizar

**Verificação:**
```bash
sudo kubectl get pods -n estudo -o wide | grep -E 'ml-workload|generic'
```

---

## Exercício 6: Kind — Cluster Multi-Node Local

**Contexto:** Você precisa testar um manifesto que usa `nodeAffinity` e `podAntiAffinity` antes de aplicar em produção. O homelab tem apenas 2 workers, mas você precisa de pelo menos 3 para a demonstração. Kind resolve isso localmente no WSL2.

**Missão:** Criar um cluster Kind multi-node, testar distribuição de Pods, e carregar uma imagem local.

**Requisitos:**
- [ ] Instalar Kind no WSL2 (se não instalado)
- [ ] Criar um cluster Kind chamado `lab` com 1 control-plane e **3 workers** usando um arquivo de configuração
- [ ] Verificar que os 4 nodes aparecem como `Ready`
- [ ] Criar um Deployment com `podAntiAffinity` `requiredDuringSchedulingIgnoredDuringExecution` para que cada réplica vá para um node diferente — testar com 3 réplicas (deve funcionar) e depois 4 réplicas (deve travar em Pending)
- [ ] Fazer pull da imagem `nginx:alpine` localmente e carregá-la no Kind sem usar um registry
- [ ] Deletar o cluster ao finalizar

**Verificação:**
```bash
kind get clusters
kubectl get nodes --context kind-lab
kubectl get pods -o wide   # verificar distribuição entre nodes
```

---

## Exercício 7: Kind com CNI Customizado (Calico)

**Contexto:** Você precisa testar uma NetworkPolicy antes de aplicar em produção. O CNI padrão do Kind (kindnet) **não** suporta NetworkPolicy. Precisa trocar para Calico.

**Missão:** Criar um cluster Kind com Calico CNI e validar que NetworkPolicy funciona.

**Requisitos:**
- [ ] Criar um cluster Kind com `disableDefaultCNI: true` e `podSubnet: 192.168.100.0/24`
- [ ] Verificar que os nodes ficam em `NotReady` (CNI ausente)
- [ ] Instalar Calico no cluster Kind
- [ ] Aguardar todos os nodes ficarem `Ready` e o Calico estar operacional
- [ ] Criar dois Pods (`frontend` e `backend`) e uma NetworkPolicy que:
  - Bloqueia **todo** ingress no Pod `backend` por padrão
  - Permite ingress apenas de Pods com label `app: frontend`
- [ ] Verificar: `frontend → backend` funciona, `kubectl exec` em um terceiro Pod sem label `app: frontend` → `backend` falha

**Verificação:**
```bash
kubectl get nodes --context kind-calico
kubectl get pods -n kube-system | grep calico

# Teste de conectividade
kubectl exec -it frontend -- wget -qO- http://backend:80
kubectl exec -it bloqueado -- wget --timeout=5 -qO- http://backend:80
# O segundo deve dar timeout
```

> [!tip] Versão do Calico para Kind
> Use o manifesto simples do Calico (não o Tigera Operator) para Kind — é mais leve e sem dependências de CRDs extras. O manifest `calico.yaml` direto do repositório funciona melhor em ambientes Kind.

---

## Exercício 8: Simular Upgrade de Cluster (Staff-level)

**Contexto:** O cluster de produção está na v1.35. A próxima versão disponível é v1.36. O processo de upgrade deve ser executado node por node, sem downtime de workloads. Você vai simular o processo completo em um cluster Kind antes de executar em produção.

**Missão:** Documentar e executar o processo completo de upgrade em um cluster Kind, validando cada etapa.

**Requisitos:**

**Parte A — Preparação:**
- [ ] Criar um cluster Kind com a versão **anterior** do Kubernetes (ex: v1.34 — verificar se disponível como imagem Kind)
- [ ] Fazer o deploy de um Deployment de teste com 3 réplicas e `podAntiAffinity` para garantir distribuição
- [ ] Verificar a versão atual: `kubectl version`

**Parte B — Upgrade do Control-Plane:**
- [ ] Instalar a versão alvo do `kubeadm` no container do control-plane Kind
- [ ] Executar `kubeadm upgrade plan` e verificar o plano
- [ ] Executar `kubeadm upgrade apply` para o control-plane
- [ ] Verificar que os componentes do control-plane estão na nova versão

**Parte C — Upgrade dos Workers (um por vez):**
- [ ] Fazer drain do primeiro worker
- [ ] Verificar que os Pods do Deployment foram reagendados nos outros nodes
- [ ] Instalar nova versão do kubelet no worker e reiniciar
- [ ] Fazer uncordon e verificar que o node volta como `Ready` na nova versão
- [ ] Repetir para os demais workers

**Parte D — Validação:**
- [ ] Confirmar que todos os nodes estão na nova versão
- [ ] Confirmar que o Deployment continua funcionando com as 3 réplicas
- [ ] Documentar os passos que dariam diferente em produção (LB externo, etcd backup, janela de manutenção)

**Verificação:**
```bash
kubectl get nodes   # todos na nova versão
kubectl get pods -o wide   # Deployment sem interruções
kubeadm version && kubectl version && kubelet --version
```

> [!warning] Em Produção
> Sempre fazer backup do etcd antes de um upgrade:
> `sudo ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
>   --endpoints=https://127.0.0.1:2379 \
>   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
>   --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
>   --key=/etc/kubernetes/pki/etcd/healthcheck-client.key`
