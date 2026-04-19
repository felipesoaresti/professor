---
tags:
  - kubernetes
  - kubeadm
  - cni
  - taints
  - kind
area: kubernetes
tipo: conteudo
prerequisites:
  - "[[05-Kubernetes/kubernetes-teoria-inicial]]"
  - "[[05-Kubernetes/labels-annotations]]"
next:
  - "[[11-Exercicios/cluster-kubeadm-cni-kind]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Cluster Kubernetes — kubeadm, CNI, Taints e Kind

> Exercícios: [[11-Exercicios/cluster-kubeadm-cni-kind]] | Trilha: [[00-Trilha/kubernetes]]

---

## O que é e por que existe

O Kubernetes não instala a si mesmo. Alguém precisa:
1. Gerar certificados para TLS entre os componentes (apiserver, etcd, kubelet, controller-manager, scheduler)
2. Escrever os kubeconfigs com os endpoints e credenciais corretos
3. Criar os static pods do control-plane (apiserver, etcd, controller-manager, scheduler) como manifests em `/etc/kubernetes/manifests/`
4. Configurar o kubelet nos workers para apontar para o apiserver
5. Plugar uma rede (CNI) para que Pods se comuniquem entre nodes

**kubeadm** automatiza exatamente esses passos. É a ferramenta oficial de bootstrapping — não é um instalador de produção completo (não configura HA avançado, LB externo, monitoring), mas é o bloco base sobre o qual ferramentas como Cluster API, Rancher e kops se constroem.

**Kind** (Kubernetes in Docker) sobe um cluster inteiro dentro de containers Docker — cada "node" é um container. Serve para desenvolvimento local e CI.

**Taints e Tolerations** controlam quais Pods podem ser agendados em quais nodes — mecanismo de exclusão. NodeAffinity é atração, Taints é repulsão.

---

## Como funciona internamente

### kubeadm — Fases de Inicialização

`kubeadm init` executa fases sequenciais, cada uma auditável individualmente:

```
preflight          → verifica pré-requisitos (swap off, portas livres, cgroups)
certs              → gera CA + certificados para todos os componentes
kubeconfig         → gera admin.conf, controller-manager.conf, scheduler.conf, kubelet.conf
kubelet-start      → escreve config do kubelet e inicia o serviço
control-plane      → cria manifests dos static pods em /etc/kubernetes/manifests/
etcd               → cria manifest do etcd (se não externo)
upload-config      → salva ClusterConfiguration no ConfigMap kubeadm-config (namespace kube-system)
upload-certs       → cifra e salva certificados no Secret kubeadm-certs (para HA join)
mark-control-plane → adiciona labels e taints no node control-plane
bootstrap-token    → cria token de join (TTL 24h por padrão)
kubelet-finalize   → configura kubelet para usar PKI do cluster
addons             → instala CoreDNS e kube-proxy
```

Para ver e rodar fases individualmente:
```bash
kubeadm init --dry-run
kubeadm init phase certs all
kubeadm init phase control-plane all
```

### Árvore de Certificados

```
/etc/kubernetes/pki/
├── ca.crt / ca.key                    ← CA raiz do cluster
├── apiserver.crt / apiserver.key      ← TLS do apiserver (inclui SAN: IP, DNS, kubernetes.default.svc)
├── apiserver-kubelet-client.*         ← apiserver → kubelet (mTLS)
├── apiserver-etcd-client.*            ← apiserver → etcd
├── front-proxy-ca.*                   ← CA do aggregation layer
├── front-proxy-client.*
└── etcd/
    ├── ca.crt / ca.key                ← CA separada do etcd
    ├── server.* / peer.* / healthcheck-client.*
```

Certificados têm validade de **1 ano** (exceto a CA, que tem 10 anos). O `kubeadm certs check-expiration` mostra todas as datas.

### Static Pods — Como o Control-Plane Sobe Sem o Apiserver

O kubelet no master monitora `/etc/kubernetes/manifests/`. Quando encontra um YAML lá, cria o container diretamente via containerd — sem passar pelo apiserver. Isso resolve o problema do galinha-e-ovo: o apiserver é um static pod, então o kubelet o sobe antes de qualquer apiserver existir.

```
/etc/kubernetes/manifests/
├── etcd.yaml
├── kube-apiserver.yaml
├── kube-controller-manager.yaml
└── kube-scheduler.yaml
```

Editar esses arquivos diretamente causa restart automático do respectivo componente em segundos.

### kubeadm join — Workers e Control-Plane Adicional

O `kubeadm join` em um worker:
1. Valida o token de bootstrap com a CA thumbprint (evita man-in-the-middle)
2. Cria um CSR (Certificate Signing Request) para o kubelet
3. O controller-manager aprova automaticamente (via bootstrap RBAC) e emite o certificado
4. O kubelet usa o certificado para se autenticar no apiserver

Para adicionar um segundo control-plane (HA):
```bash
# Primeiro, fazer upload dos certs do primeiro master
kubeadm init phase upload-certs --upload-certs
# Retorna: --certificate-key <hash>

# No segundo master:
kubeadm join <apiserver>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key>
```

### ClusterConfiguration — O Arquivo de Config

Em vez de passar flags na linha de comando, o recomendado é um arquivo YAML:

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.35.0
clusterName: homelab
controlPlaneEndpoint: "192.168.3.30:6443"   # IP do LB ou do primeiro master
networking:
  podSubnet: "10.244.0.0/16"               # deve ser compatível com o CNI escolhido
  serviceSubnet: "10.96.0.0/12"
  dnsDomain: "cluster.local"
apiServer:
  extraArgs:
    audit-log-path: /var/log/audit.log
    audit-log-maxage: "30"
  certSANs:
    - "192.168.3.30"
    - "k8s-master"
    - "kubernetes.staypuff.info"
etcd:
  local:
    dataDir: /var/lib/etcd
controllerManager:
  extraArgs:
    bind-address: "0.0.0.0"               # necessário para Prometheus scraping
scheduler:
  extraArgs:
    bind-address: "0.0.0.0"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.168.3.30"
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  kubeletExtraArgs:
    node-ip: "192.168.3.30"
```

---

## CNI — Escolha de Rede

### O que é CNI

CNI (Container Network Interface) é uma **especificação** — define uma interface JSON que o kubelet chama para configurar a rede de cada Pod. O kubelet não sabe nada de roteamento; delega 100% ao plugin CNI.

O plugin CNI recebe: namespace de rede do container, nome do Pod, namespace K8s, e deve retornar o IP alocado e as rotas configuradas.

Arquivos relevantes:
```
/etc/cni/net.d/          ← configuração do plugin (JSON)
/opt/cni/bin/            ← binários dos plugins
```

### Como Funciona a Comunicação entre Pods em Nodes Diferentes

Cada node tem uma subnet do `--pod-network-cidr` alocada para seus Pods. O CNI precisa garantir que:
- Pod no worker1 (10.244.1.0/24) consiga alcançar Pod no worker2 (10.244.2.0/24)
- Sem NAT entre Pods (requisito do modelo de rede Kubernetes)

Cada CNI resolve isso de forma diferente:

| CNI | Mecanismo de Roteamento | NetworkPolicy | Performance | Complexidade |
|---|---|---|---|---|
| **Flannel** | VXLAN overlay (UDP encapsulation) | Não nativa | Moderada | Mínima |
| **Calico** | BGP ou VXLAN, sem encapsulation em L3 flat | Sim (nativa) | Alta (BGP) | Média |
| **Cilium** | eBPF (bypass iptables completamente) | Sim + L7 | Muito alta | Alta |
| **Weave** | Mesh overlay, gossip protocol | Sim | Moderada | Média |
| **Canal** | Flannel + Calico NetworkPolicy | Sim | Moderada | Média |

### Flannel — Simples, sem NetworkPolicy

Usa VXLAN por padrão: encapsula frames Ethernet em pacotes UDP. Todo tráfego inter-node passa por um tunnel. Overhead de ~50 bytes por pacote.

Não implementa NetworkPolicy — se precisar de NetworkPolicy com Flannel, precisa do Canal (Flannel + Calico).

```bash
# Pod CIDR deve ser 10.244.0.0/16 (padrão do Flannel)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### Calico — BGP, NetworkPolicy, Homelab

O homelab usa Calico. Por padrão em redes L2 (todos os nodes no mesmo switch), Calico usa **BGP com full mesh**: cada node é um BGP peer de todos os outros. Não há encapsulation — pacotes entre Pods passam pela rede física como qualquer rota IP normal.

Em ambientes onde BGP não é possível (cloud, VLANs múltiplas), Calico cai para VXLAN ou IPIP.

```bash
# Instalar Operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/tigera-operator.yaml

# Configurar com o pod CIDR correto
kubectl create -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet  # VXLAN só quando necessário
      natOutgoing: Enabled
      nodeSelector: all()
EOF
```

### Cilium — eBPF, o Estado da Arte

Cilium substitui kube-proxy e o próprio iptables por programas eBPF carregados diretamente no kernel. O resultado: menor latência, maior throughput, e visibilidade L7 (consegue ver URLs HTTP, queries SQL) sem sidecar.

Requer kernel ≥ 4.9 (recomendado ≥ 5.10). Complexo de operar, mas é o caminho das clouds modernas (GKE Dataplane V2, EKS).

```bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.16.0 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=192.168.3.30 \
  --set k8sServicePort=6443
```

### Como Escolher

```
Precisa de NetworkPolicy?
  ├── Não → Flannel (simples, estável)
  └── Sim →
        Kernel ≥ 5.10 e quer visibilidade L7?
          ├── Sim → Cilium
          └── Não →
                BGP disponível na rede?
                  ├── Sim → Calico (BGP sem overlay)
                  └── Não → Calico (VXLAN) ou Weave
```

---

## Taints e Tolerations

### Conceito

**Taint** é aplicado no **node** — "este node repele Pods sem toleração para esta marca".
**Toleration** é aplicado no **Pod** — "este Pod tolera aquela marca".

Sem toleration, o scheduler não coloca o Pod no node tainted. Com toleration, o Pod **pode** ir para lá — mas não é obrigado (para forçar, use nodeAffinity ou nodeSelector junto).

### Anatomy

```yaml
# Taint no node
key=value:effect

# Effect pode ser:
NoSchedule          # Não agenda novos Pods sem toleration — Pods existentes continuam
PreferNoSchedule    # Tenta não agendar, mas não é garantia
NoExecute           # Não agenda E remove Pods existentes sem toleration (com grace period)
```

```bash
# Adicionar taint
kubectl taint nodes worker1 dedicated=gpu:NoSchedule

# Remover taint (sufixo -)
kubectl taint nodes worker1 dedicated=gpu:NoSchedule-

# Ver taints de todos os nodes
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

### Toleration no Pod

```yaml
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  
  # Tolerar qualquer taint com essa key (sem checar value)
  - key: "dedicated"
    operator: "Exists"
    effect: "NoSchedule"
  
  # Tolerar QUALQUER taint (perigoso — só para DaemonSets específicos)
  - operator: "Exists"
```

### Control-Plane Taint — Por que o Master não Roda Pods de Usuário

Ao inicializar com kubeadm, o master recebe automaticamente:
```
node-role.kubernetes.io/control-plane:NoSchedule
```

Isso impede que workloads de usuário sejam agendados no control-plane. Para ambientes de desenvolvimento de um único node (remover o isolamento):
```bash
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule-
```

### NoExecute + tolerationSeconds — Eviction Controlada

```yaml
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300    # aguarda 5 minutos antes de ser removido do node not-ready
```

O Kubernetes adiciona automaticamente tolerations para `not-ready` e `unreachable` com 300s em todos os Pods. Isso significa que um node que fica offline leva 5 minutos para ter seus Pods removidos e reagendados em outro node.

Para aplicações stateless que devem reagendar rápido:
```yaml
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 30     # reagenda em 30s
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 30
```

### Taints Automáticos do Kubernetes

O node lifecycle controller adiciona taints automaticamente quando detecta problemas:

| Condição | Taint Automático |
|---|---|
| Node not ready | `node.kubernetes.io/not-ready:NoExecute` |
| Node unreachable | `node.kubernetes.io/unreachable:NoExecute` |
| Disk pressure | `node.kubernetes.io/disk-pressure:NoSchedule` |
| Memory pressure | `node.kubernetes.io/memory-pressure:NoSchedule` |
| PID pressure | `node.kubernetes.io/pid-pressure:NoSchedule` |
| Network unavailable | `node.kubernetes.io/network-unavailable:NoSchedule` |
| Node unschedulable | `node.kubernetes.io/unschedulable:NoSchedule` |
| CNI não inicializado | `node.kubernetes.io/not-ready:NoSchedule` (temporário) |

### Taint vs NodeAffinity vs nodeSelector

| Mecanismo | Direção | Tipo | Uso |
|---|---|---|---|
| `nodeSelector` | Pod atrai node | Hard (obrigatório) | Scheduling simples por label |
| `nodeAffinity` | Pod atrai/evita node | Hard ou Soft | Scheduling expressivo (In, NotIn, Exists) |
| `taint/toleration` | Node repele Pod | Hard ou Soft (PreferNoSchedule) | Reservar nodes, excluir workloads |
| `podAffinity` | Pod atrai outro Pod | Hard ou Soft | Co-localização (cache perto do app) |
| `podAntiAffinity` | Pod evita outro Pod | Hard ou Soft | HA (evitar single-node SPOF) |

Em produção, Taints + Tolerations + NodeAffinity são usados juntos: o taint garante que apenas Pods autorizados entrem no node, e o nodeAffinity garante que esses Pods **prefiram** (ou exijam) aquele node específico.

---

## Kind — Kubernetes in Docker

### Como Funciona

Kind cria um container Docker para cada node Kubernetes. Dentro de cada container:
- Roda um processo `systemd` (ou similar) como PID 1
- O kubelet executa dentro do container
- containerd roda dentro do container (Docker-in-Docker)
- O CNI padrão é **kindnet** (baseado em bridge + rotas estáticas)

O kubeconfig gerado aponta para uma porta mapeada no `localhost` — transparente para `kubectl`.

### Instalação

```bash
# Linux/WSL2
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.25.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
```

### Cluster Básico

```bash
# Cluster com 1 node (control-plane + worker)
kind create cluster --name meu-cluster

# Verificar nodes
kubectl get nodes

# Deletar
kind delete cluster --name meu-cluster
```

### Cluster Multi-Node com Configuração

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: lab
nodes:
- role: control-plane
  # Mapear porta para usar Ingress no localhost
  extraPortMappings:
  - containerPort: 80
    hostPort: 8080
    protocol: TCP
  - containerPort: 443
    hostPort: 8443
    protocol: TCP
- role: worker
- role: worker
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  disableDefaultCNI: false   # true se quiser instalar Calico/Cilium manualmente
```

```bash
kind create cluster --config kind-config.yaml
```

### Usar CNI Diferente no Kind

```yaml
# kind-config-calico.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true   # desabilita kindnet
  podSubnet: "10.244.0.0/16"
nodes:
- role: control-plane
- role: worker
```

```bash
kind create cluster --config kind-config-calico.yaml
# Instalar Calico após o cluster subir (nodes ficam NotReady até o CNI estar presente)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/calico.yaml
```

### Carregar Imagens Locais no Kind

Kind não acessa o registry local do Docker automaticamente:
```bash
# Build local
docker build -t minha-app:dev .

# Carregar no kind
kind load docker-image minha-app:dev --name lab

# No Pod, usar imagePullPolicy: Never ou IfNotPresent
```

### Kind vs Minikube vs k3d

| | Kind | Minikube | k3d |
|---|---|---|---|
| Runtime | Docker containers | VM ou Docker | k3s em Docker |
| Multi-node | Sim | Sim (parcial) | Sim |
| Fidelidade ao K8s | Alta (kubeadm real) | Média | Média (k3s é modificado) |
| Velocidade de criação | ~30s | ~2-3min | ~15s |
| Uso de memória | Baixo | Alto | Muito baixo |
| CI/CD | Ideal | Possível | Bom |
| Limitações | Sem LoadBalancer real | Storage mais simples | k3s ≠ upstream |

---

## Na prática — Comandos e Exemplos Reais

### Inicializar Cluster do Zero (Debian 13)

```bash
# Pré-requisitos em TODOS os nodes
swapoff -a && sed -i '/swap/d' /etc/fstab

modprobe overlay
modprobe br_netfilter

cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# Instalar containerd
apt install -y containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
# Mudar SystemdCgroup = true em /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd

# Instalar kubeadm, kubelet, kubectl
apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' > /etc/apt/sources.list.d/kubernetes.list
apt-get update && apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# No master: inicializar
kubeadm init --config kubeadm-config.yaml

# Configurar kubectl
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config

# Instalar CNI (Calico)
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/tigera-operator.yaml

# Nos workers: join
kubeadm join 192.168.3.30:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### Gerenciar Certificados

```bash
# Ver validade de todos os certificados
kubeadm certs check-expiration

# Renovar todos (antes de expirar ou após expirar)
kubeadm certs renew all

# Após renovar, reiniciar os static pods
# Mover e devolver os manifests faz o kubelet recriar os containers
cd /etc/kubernetes/manifests
mv kube-apiserver.yaml /tmp/ && sleep 5 && mv /tmp/kube-apiserver.yaml .
```

### Upgrade de Versão

```bash
# No master
apt-get install -y kubeadm=1.36.0-1.1
kubeadm upgrade plan
kubeadm upgrade apply v1.36.0

apt-get install -y kubelet=1.36.0-1.1 kubectl=1.36.0-1.1
systemctl daemon-reload && systemctl restart kubelet

# Nos workers (um por vez)
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data
# No worker1:
apt-get install -y kubeadm=1.36.0-1.1
kubeadm upgrade node
apt-get install -y kubelet=1.36.0-1.1
systemctl daemon-reload && systemctl restart kubelet
# No master:
kubectl uncordon worker1
```

### Casos de Uso de Taints

```bash
# Node dedicado para GPU workloads
kubectl taint nodes gpu-node accelerator=nvidia:NoSchedule
kubectl label nodes gpu-node accelerator=nvidia

# No Pod GPU:
spec:
  tolerations:
  - key: "accelerator"
    operator: "Equal"
    value: "nvidia"
    effect: "NoSchedule"
  nodeSelector:
    accelerator: nvidia

# Preparar node para manutenção (drain)
kubectl cordon worker2          # marca como Unschedulable (taint implícito)
kubectl drain worker2 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=120
# ... manutenção ...
kubectl uncordon worker2
```

---

## Troubleshooting — Cenários Reais de Produção

### Node fica NotReady após instalação — CNI não instalado

```bash
kubectl get nodes
# STATUS: NotReady
# Motivo: CNI não configurado

kubectl describe node k8s-master | grep -A5 Conditions
# KubeletNotReady: container runtime network not ready:
# NetworkReady=false reason:NetworkPluginNotReady

# Solução: instalar o CNI escolhido
```

### kubeadm init falha — swap habilitado

```bash
[preflight] ERROR: /proc/swaps shows that swap is enabled
# Solução:
swapoff -a
# E remover do /etc/fstab para persistir após reboot
```

### Token de join expirado (TTL 24h)

```bash
# No master, gerar novo token
kubeadm token create --print-join-command
# Ou listar tokens existentes
kubeadm token list
```

### Certificado do apiserver não inclui novo IP/DNS

```bash
# Erro no kubectl: x509: certificate is valid for X, not Y
# Solução: regenerar certificado do apiserver com o novo SAN

# Adicionar o novo SAN ao ClusterConfiguration no ConfigMap
kubectl -n kube-system edit cm kubeadm-config
# Editar certSANs no apiServer

# Mover o certificado atual (backup)
mv /etc/kubernetes/pki/apiserver.{crt,key} /tmp/

# Regenerar
kubeadm init phase certs apiserver --config /etc/kubernetes/kubeadm-config.yaml

# Reiniciar apiserver
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 5
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

### Pod em Pending com "0/3 nodes are available: 3 node(s) had untolerated taint"

```bash
kubectl describe pod <pod> | grep -A10 Events
# 0/3 nodes are available: 1 node(s) had untolerated taint {dedicated: gpu}

# Ver taints de todos os nodes
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Opções:
# 1. Adicionar toleration ao Pod
# 2. Remover taint do node
# 3. Usar nodeAffinity diferente
```

### Kind — Pods não se comunicam entre nodes

```bash
# Verificar se o CNI está rodando
kubectl get pods -n kube-system | grep -E 'kindnet|calico|flannel'

# Se usando CNI customizado com disableDefaultCNI: true e esqueceu de instalar:
kubectl get nodes
# Todos NotReady — instalar o CNI escolhido
```

---

## Nível Avançado — Edge Cases e CKA/Staff

### kubeadm init --skip-phases

Para clusters que precisam de customização específica em uma fase:
```bash
# Gerar os manifests sem iniciar os componentes
kubeadm init phase control-plane all --config config.yaml

# Editar os manifests gerados antes de startarem
vi /etc/kubernetes/manifests/kube-apiserver.yaml

# Continuar com as fases seguintes
kubeadm init phase etcd local --config config.yaml
kubeadm init phase mark-control-plane
kubeadm init phase bootstrap-token
kubeadm init phase addon all
```

### etcd Externo — HA Avançado

Por padrão, kubeadm sobe um etcd local (stacked). Para produção séria, o etcd pode ser separado em um cluster próprio de 3 ou 5 nodes:

```yaml
# ClusterConfiguration
etcd:
  external:
    endpoints:
    - "https://etcd1:2379"
    - "https://etcd2:2379"
    - "https://etcd3:2379"
    caFile: /etc/etcd/ca.crt
    certFile: /etc/etcd/apiserver-etcd-client.crt
    keyFile: /etc/etcd/apiserver-etcd-client.key
```

Isso permite escalar apiserver independentemente do etcd e evita que uma falha do apiserver corrumpa o etcd (split-brain em stacked).

### Calico BGPPeer — Peering com Roteador Físico

No homelab, Calico usa BGP full-mesh entre nodes. Em redes corporativas, é possível fazer peering com o roteador físico (ToR switch) para que as rotas dos Pods apareçam no roteamento físico da rede — sem overlay, sem encapsulation, com latência mínima:

```yaml
apiVersion: crd.projectcalico.org/v1
kind: BGPPeer
metadata:
  name: roteador-principal
spec:
  peerIP: 192.168.3.1
  asNumber: 65001
```

### TaintBasedEvictions — Controle Fino de Failover

O comportamento padrão de eviction (300s) pode ser customizado por workload:

```yaml
# StatefulSet de banco de dados — não reagendar rápido, esperar o node voltar
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 3600   # espera 1 hora

# Frontend stateless — reagendar imediatamente
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 10
```

### CNI Migration — Trocar CNI em Cluster Vivo

Trocar CNI sem downtime é possível mas complexo:
1. Instalar novo CNI em paralelo (diferente `podSubnet`)
2. Fazer drain e uncordon de um node por vez
3. Verificar conectividade após cada node
4. Remover CNI antigo por último

O risco real: durante a migração, Pods no CNI antigo e Pods no novo CNI podem não conseguir se comunicar (subnets diferentes, policies diferentes).

### Kind em CI/CD — GitHub Actions

```yaml
# .github/workflows/e2e.yaml
- name: Create Kind cluster
  uses: helm/kind-action@v1.10.0
  with:
    cluster_name: ci-cluster
    config: .github/kind-config.yaml

- name: Load local image
  run: kind load docker-image minha-app:${{ github.sha }} --name ci-cluster

- name: Run e2e tests
  run: kubectl apply -f tests/manifests/ && kubectl wait --for=condition=ready pod -l app=test
```

---

## Referências

- [kubeadm reference](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) — documentação oficial completa
- [Creating HA clusters](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/) — stacked etcd vs external etcd
- [CNI spec](https://github.com/containernetworking/cni) — especificação oficial
- [Calico docs](https://docs.tigera.io/calico/latest/about/) — BGP, NetworkPolicy, eBPF mode
- [Cilium docs](https://docs.cilium.io/) — eBPF, Hubble observability
- [Kind docs](https://kind.sigs.k8s.io/) — configuração avançada, LoadBalancer com MetalLB
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) — spec completa
- DK8S Day1-Day3 — `/mnt/c/Users/felip/Documents/Kubernets/DK8S`
