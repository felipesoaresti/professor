---
tags:
  - networking
  - kubernetes
  - calico
  - cni
  - bgp
  - networkpolicy
area: networking
tipo: conteudo
prerequisites:
  - "[[05-Kubernetes/kubernetes-teoria-inicial]]"
  - "[[05-Kubernetes/cluster-kubeadm-cni-kind]]"
next:
  - "[[11-Exercicios/calico-network]]"
trilha: "[[00-Trilha/networking]]"
---

# Calico Network

> Exercícios: [[11-Exercicios/calico-network]] | Trilha: [[00-Trilha/networking]]

---

## O que é e por que existe

O modelo de rede do Kubernetes impõe três regras:
1. Todo Pod pode alcançar qualquer outro Pod **sem NAT**
2. Nodes podem alcançar qualquer Pod **sem NAT**
3. O IP que um Pod vê como próprio é o mesmo IP que outros veem

Essas regras não existem no kernel por padrão. Precisam ser implementadas — e é isso que um CNI faz.

**Calico** resolve o problema através de **roteamento L3 puro**: cada node anuncia via BGP as subnets dos Pods que hospeda. Quando um Pod no worker1 manda um pacote para um Pod no worker2, o kernel do worker1 simplesmente roteia o pacote pelo gateway normal da rede — sem encapsulation, sem tunnel, sem overhead.

O resultado é rede com latência de rede física, debugging via `ip route` e `tcpdump` normais, e NetworkPolicy com semântica de firewall real implementada via iptables ou eBPF.

**Por que não Flannel?** Flannel encapsula tráfego inter-node em VXLAN (UDP). Simples, mas: overhead de CPU, payload menor (MTU -50 bytes), sem NetworkPolicy nativa, debugging opaco.

**Por que não Cilium?** Cilium usa eBPF e supera Calico em performance e visibilidade L7. Mas requer kernel ≥ 5.10, é mais complexo e tem curva de operação maior. Para a maioria dos clusters on-premise, Calico é o ponto ótimo entre poder e complexidade.

---

## Como funciona internamente

### Componentes

```
┌─────────────────────────────────────────────────────────┐
│                    kube-apiserver                        │
│              (Calico usa API K8s como datastore)         │
└────────────────────────┬────────────────────────────────┘
                         │ watch
          ┌──────────────┼──────────────┐
          │              │              │
   ┌──────▼─────┐ ┌──────▼──────┐ ┌───▼──────────────────┐
   │   Typha    │ │calico-kube- │ │   calico-node          │
   │ (cache/fan-│ │controllers  │ │   (DaemonSet — 1/node) │
   │  out proxy)│ │(sync K8s→   │ ├────────────────────────┤
   └──────┬─────┘ │ Calico CRDs)│ │ Felix    │ BIRD │confd │
          │       └─────────────┘ │ (iptables │ (BGP)│(conf)│
          │ watch                 │  ou eBPF) │      │      │
   ┌──────▼───────┐               └──────┬───┴──────┴──────┘
   │ calico-nodes │                      │
   │ (todos)      │               ┌──────▼──────────────────┐
   └──────────────┘               │  kernel (iptables/eBPF) │
                                  │  + tabela de roteamento  │
                                  └─────────────────────────┘
```

**Felix** — o agente mais crítico. Roda em cada node e é responsável por:
- Programar regras de iptables (ou eBPF) para implementar NetworkPolicy
- Programar rotas do kernel para os Pods locais
- Reportar o estado dos endpoints para o datastore

**BIRD** (BGP Internet Routing Daemon) — anuncia as rotas dos Pods de cada node para os outros nodes via BGP. É um daemon BGP de produção completo, o mesmo usado em roteadores de ISPs.

**confd** — monitora o datastore Calico e regenera a configuração do BIRD quando algo muda (novo node, novo IPPool, etc.).

**Typha** — intermediário entre o API server e múltiplos calico-nodes. Em clusters pequenos (<50 nodes) é opcional; em clusters grandes, evita que cada calico-node faça watch direto no apiserver (que multiplicaria a carga N vezes).

**calico-kube-controllers** — sincroniza recursos do Kubernetes (Pods, Namespaces, NetworkPolicy, Nodes) para os CRDs do Calico, mantendo o modelo de dados Calico consistente com o estado do cluster.

### Data Plane — Dois Caminhos Possíveis

**iptables (padrão):**
- Felix insere chains no iptables: `cali-FORWARD`, `cali-INPUT`, `cali-OUTPUT`, `cali-pi-*` (policy ingress), `cali-po-*` (policy output)
- Cada endpoint (interface de Pod) tem um par de chains: `cali-tw-<hash>` (to workload) e `cali-fw-<hash>` (from workload)
- Performance degrada linearmente com número de regras — em clusters com >10k policies, pode ser gargalo

**eBPF (opt-in):**
- Felix carrega programas eBPF diretamente nas interfaces de rede
- Bypass total do iptables — menor latência, menor overhead de CPU
- Requer kernel ≥ 5.3 (recomendado ≥ 5.10)
- Habilitar: `calicoctl patch felixconfiguration default --patch='{"spec": {"bpfEnabled": true}}'`

### Ciclo de Vida de um Pacote inter-node (modo BGP, sem overlay)

```
Pod A (10.244.1.5) no worker1 → Pod B (10.244.2.8) no worker2

1. Pod A envia pacote destino 10.244.2.8
2. Interface veth do Pod A conecta ao namespace do host
3. Felix programou: via interface eth0 → gateway 192.168.3.1 (ou direto para 192.168.3.32)
4. BIRD no worker1 anunciou via BGP: "10.244.2.0/26 está em 192.168.3.32"
5. Kernel do worker1 encaminha pacote para 192.168.3.32 (IP do worker2)
6. Kernel do worker2 recebe pacote, vê destino 10.244.2.8
7. Rota local: 10.244.2.8 está na interface veth do Pod B
8. Pacote entrega ao Pod B — IP origem 10.244.1.5 preservado (sem NAT)
```

---

## IPAM — IP Address Management

### IPPool

O IPPool define o bloco de IPs do qual todos os Pods do cluster recebem endereços:

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 10.244.0.0/16         # bloco total
  blockSize: 26               # cada node recebe um /26 (64 IPs)
  ipipMode: Never             # sem IPIP overlay
  vxlanMode: Never            # sem VXLAN overlay
  natOutgoing: true           # SNAT para tráfego saindo do cluster
  nodeSelector: all()         # todos os nodes usam este pool
  disabled: false
```

**blockSize:** o Calico aloca blocos de IPs por node. Com blockSize=26, cada node tem até 64 IPs de Pod. Com 3 nodes e /16: teóricamente 1024 blocos × 64 = 65536 IPs — muito mais que suficiente.

Para ver alocação atual:
```bash
calicoctl ipam show --show-blocks
```

### IPReservation

Para excluir IPs específicos de serem alocados para Pods (ex: IPs já usados por VMs na mesma subnet):
```yaml
apiVersion: projectcalico.org/v3
kind: IPReservation
metadata:
  name: reservas-homelab
spec:
  reservedCIDRs:
  - "10.244.0.0/26"   # bloco reservado para serviços de infraestrutura
```

### natOutgoing

Quando um Pod acessa a internet (ex: `curl google.com`), o tráfego sai com o IP do Pod (10.244.x.x) — que não é roteável na internet. `natOutgoing: true` faz SNAT para o IP do node antes de sair da interface física. É o padrão e é o comportamento esperado.

---

## Modos de Roteamento

### BGP Full Mesh (padrão — clusters pequenos)

Cada node estabelece sessão BGP com **todos** os outros nodes. Com 3 nodes, são 3 sessões BGP (par a par). Com 100 nodes, são 4950 sessões — inviável.

```bash
# Ver estado das sessões BGP
sudo calicoctl node status
# Mostra peers, estado (Established/Idle), e prefixos recebidos
```

### Route Reflectors (clusters grandes — >50 nodes)

Em vez de N×N sessões, alguns nodes atuam como Route Reflectors (RR): recebem rotas de todos e redistribuem. Cada node só precisa de N_RR sessões:

```yaml
# Designar um node como Route Reflector
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  name: k8s-master
spec:
  bgp:
    routeReflectorClusterID: "1.0.0.1"   # qualquer IP fictício como cluster ID
    routeReflectorClient: true
```

### IPIP (IP-in-IP Overlay)

Quando nodes estão em subnets diferentes (cloud multi-AZ, VLANs separadas), o BGP pode anunciar rotas mas o roteador intermediário não sabe como encaminhar para 10.244.x.x. IPIP encapsula o pacote do Pod dentro de um pacote IP normal:

```
Outer: src=192.168.3.31 dst=192.168.3.32  (IPs dos nodes)
Inner: src=10.244.1.5   dst=10.244.2.8    (IPs dos Pods)
```

```yaml
spec:
  ipipMode: Always        # sempre IPIP
  ipipMode: CrossSubnet   # IPIP só quando cross-subnet, nativo quando mesmo subnet
```

`CrossSubnet` é a configuração ideal para ambientes híbridos: usa roteamento direto (mais rápido) quando possível e IPIP quando necessário.

### VXLAN

Similar ao IPIP mas encapsula em UDP/4789. Funciona em redes que bloqueiam protocolo IPIP (algumas nuvens). Pode ser usado sem BGP — o Calico usa tabela de encaminhamento própria em vez de BGP:

```yaml
spec:
  vxlanMode: Always
  ipipMode: Never
```

---

## NetworkPolicy

### Kubernetes NetworkPolicy (suportada pelo Calico)

A NetworkPolicy padrão do Kubernetes é **namespace-scoped** e tem semântica **additive** (só permite, nunca nega explicitamente):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-isolado
  namespace: producao
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:    # permitir DNS
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

> [!warning] Semântica de Seleção
> `from.podSelector` sem `namespaceSelector` seleciona Pods **no mesmo namespace**.
> Para selecionar Pods em outro namespace: combine `podSelector` + `namespaceSelector`.
> Para selecionar Pods em qualquer namespace: `namespaceSelector: {}` (seletor vazio = todos).

> [!info] Default Deny
> Criar uma NetworkPolicy com `podSelector: {}` (todos os Pods do namespace) e `policyTypes: [Ingress]` **sem regras ingress** = bloqueia todo ingress. Isso é o "default deny" — padrão recomendado.

### Calico NetworkPolicy — Recursos Extras

A CRD `NetworkPolicy` do Calico (diferente da K8s) adiciona:
- **`order`**: prioridade entre policies — menor número = maior prioridade
- **`action: Deny`** explícito (K8s NetworkPolicy não tem Deny, só Allow e "não tem regra")
- **`serviceAccountSelector`**: filtrar por ServiceAccount além de labels de Pod
- **`http`**: regras baseadas em método HTTP e path (requer Istio ou similar para L7)

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: negar-acesso-admin
  namespace: producao
spec:
  order: 100
  selector: app == 'backend'
  types:
  - Ingress
  ingress:
  - action: Deny
    source:
      selector: role == 'dev'
  - action: Allow
    source:
      selector: app == 'frontend'
```

### GlobalNetworkPolicy — Cluster-Wide

A principal vantagem do Calico sobre NetworkPolicy padrão: políticas que se aplicam a **todos os namespaces** ou a **interfaces do node** (HostEndpoints), sem precisar replicar a policy em cada namespace:

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-deny-all
spec:
  order: 1000         # executada por último (menor prioridade)
  selector: all()     # todos os endpoints do cluster
  types:
  - Ingress
  - Egress
  egress:
  - action: Allow     # liberar DNS cluster-wide
    protocol: UDP
    destination:
      ports: [53]
  - action: Allow
    protocol: TCP
    destination:
      ports: [53]
```

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: permitir-prometheus-scraping
spec:
  order: 10
  selector: prometheus.io/scrape == 'true'
  types:
  - Ingress
  ingress:
  - action: Allow
    source:
      selector: app.kubernetes.io/name == 'prometheus'
    ports:
    - 9090
    - 9091
    - 8080
```

### NetworkSet e GlobalNetworkSet

Agrupa CIDRs externos como entidade nomeada — evita repetir IPs em múltiplas policies:

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkSet
metadata:
  name: escritorios-vpn
  labels:
    role: trusted-network
spec:
  nets:
  - "10.10.0.0/16"     # VPN sede
  - "10.20.0.0/16"     # VPN filial
  - "192.168.3.0/24"   # homelab
```

```yaml
# Usar em GlobalNetworkPolicy
ingress:
- action: Allow
  source:
    selector: role == 'trusted-network'
```

### HostEndpoint — Protegendo o Próprio Node

HostEndpoint aplica policy nas interfaces físicas do node (eth0, ens3), protegendo o node de acessos não autorizados:

```yaml
apiVersion: projectcalico.org/v3
kind: HostEndpoint
metadata:
  name: worker1-eth0
  labels:
    node: worker1
spec:
  interfaceName: eth0
  node: k8s-worker1
  expectedIPs: ["192.168.3.31"]
```

> [!warning] HostEndpoint é perigoso sem cuidado
> Aplicar HostEndpoints sem as GlobalNetworkPolicy corretas bloqueia **todo** acesso ao node, incluindo SSH. Sempre ter uma regra Allow para SSH antes de ativar HostEndpoints.

---

## Na prática — Comandos e Exemplos Reais

### calicoctl

A CLI do Calico opera no nível dos CRDs Calico — o `kubectl` não mostra alguns recursos por nome amigável:

```bash
# Instalar calicoctl (compatível com a versão do Calico instalada)
curl -L https://github.com/projectcalico/calico/releases/download/v3.29.0/calicoctl-linux-amd64 -o calicoctl
chmod +x calicoctl && sudo mv calicoctl /usr/local/bin/

# Configurar para usar k8s API como datastore
export CALICO_DATASTORE_TYPE=kubernetes
export KUBECONFIG=/etc/kubernetes/admin.conf

# Comandos principais
calicoctl get nodes -o wide                    # nodes e BGP info
calicoctl get ippool -o yaml                   # pools de IPs
calicoctl node status                          # estado BGP do node atual
calicoctl get networkpolicy -A                 # Calico NetworkPolicies (não K8s)
calicoctl get globalnetworkpolicy              # GlobalNetworkPolicies
calicoctl get felixconfiguration -o yaml       # configuração do Felix
calicoctl ipam show                            # uso de IPs
calicoctl ipam show --show-blocks             # blocos por node
calicoctl ipam check                           # inconsistências no IPAM
```

### Inspecionar Roteamento

```bash
# Ver rotas do Calico no kernel (worker1)
ip route show | grep -v kernel
# Saída esperada:
# 10.244.2.0/26 via 192.168.3.32 dev eth0 proto bird   ← rota para Pods do worker2
# 10.244.3.0/26 via 192.168.3.30 dev eth0 proto bird   ← rota para Pods do master

# O "proto bird" confirma que a rota foi instalada pelo BIRD (BGP)

# Ver interfaces de Pods no host
ip link show | grep cali
# cali<hash>: interface veth conectada ao namespace do Pod

# Traçar o caminho de um pacote
ip route get 10.244.2.5
# 10.244.2.5 via 192.168.3.32 dev eth0 src 192.168.3.31
```

### Inspecionar Regras do Felix (iptables)

```bash
# Ver chains do Calico
sudo iptables -L -n | grep -E '^Chain cali'

# Ver regras de uma chain específica (política de um Pod)
sudo iptables -L cali-tw-<hash> -n -v

# Ver todas as regras com estatísticas (pacotes/bytes por regra)
sudo iptables -L cali-FORWARD -n -v --line-numbers

# Verificar se um pacote seria aceito/dropado (test)
sudo iptables -C cali-pi-<policy-hash> -s 10.244.1.0/26 -j ACCEPT
```

### Criar NetworkPolicy de Isolamento Completo

```yaml
# Isolar namespace estudo: bloqueia todo tráfego, libera apenas DNS e intra-namespace
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: estudo
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: permitir-dns
  namespace: estudo
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: permitir-intra-namespace
  namespace: estudo
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}    # qualquer Pod do namespace estudo
  egress:
  - to:
    - podSelector: {}
```

---

## Casos de Uso e Boas Práticas

### Estratégia de NetworkPolicy em Produção

**Modelo Pragmático (recomendado para começar):**
1. Default deny ingress por namespace
2. Allowlist explícita para cada serviço
3. DNS sempre liberado no egress
4. Monitoramento antes de deny egress (egress deny quebra muita coisa inesperadamente)

**Ordem de implementação:**
1. Implementar em namespace de não-produção primeiro
2. Observar com `kubectl get events` — Calico não loga drops por padrão
3. Habilitar packet logging do Felix para debugging antes do deny total
4. Só então aplicar em produção

### Nomear Namespaces com Labels

NetworkPolicy usa `namespaceSelector` que filtra por **labels** do namespace. Namespaces não têm labels por padrão — precisa adicionar explicitamente:

```bash
kubectl label namespace monitoring name=monitoring
kubectl label namespace kube-system name=kube-system
```

A partir do Kubernetes 1.21, o label `kubernetes.io/metadata.name` é adicionado automaticamente a todos os namespaces.

### GlobalNetworkPolicy — Regras de Baseline

Ao invés de replicar regras comuns em cada namespace, usar GlobalNetworkPolicy para:
- Permitir scraping do Prometheus em todos os namespaces
- Bloquear acesso ao metadata service da cloud (169.254.169.254)
- Permitir health checks do load balancer para todos os nodes
- Whitelist de IPs da equipe de segurança

```yaml
# Bloquear acesso ao metadata service (prevenção de SSRF em cloud)
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: bloquear-metadata-service
spec:
  order: 5
  selector: all()
  types:
  - Egress
  egress:
  - action: Deny
    destination:
      nets:
      - "169.254.169.254/32"
  - action: Allow
```

---

## Troubleshooting — Cenários Reais de Produção

### Pod não consegue alcançar outro Pod em node diferente

**Diagnóstico:**
```bash
# 1. Verificar se a rota para o bloco do Pod destino existe
ip route get <ip-do-pod-destino>
# Se não existe: problema no BGP

# 2. Verificar estado BGP
calicoctl node status
# Se peer em "Idle" ou "Active" (não "Established"): BGP não funciona

# 3. Verificar se BIRD está rodando no calico-node
kubectl exec -n kube-system <calico-node-pod> -- birdcl show protocols
# Todos peers devem estar em "Established"

# 4. Se BGP estabelecido mas sem rotas:
kubectl exec -n kube-system <calico-node-pod> -- birdcl show route
# Deve listar rotas dos outros nodes
```

**Causas comuns:**
- Firewall entre nodes bloqueando porta BGP (TCP 179)
- IP do node incorreto (verificar `calicoctl get node <name> -o yaml`)
- IPPool com `disabled: true`

### NetworkPolicy bloqueando tráfego esperado

```bash
# 1. Testar conectividade direta
kubectl exec -n estudo <pod-origem> -- wget --timeout=3 http://<pod-destino>:<porta>

# 2. Verificar quais NetworkPolicies afetam o Pod destino
kubectl get networkpolicy -n <namespace> -o yaml | grep -A5 podSelector

# 3. Habilitar logging temporário no Felix para ver drops
kubectl patch felixconfiguration default --type=merge \
  -p '{"spec":{"policySyncPathPrefix":"/var/run/nodeagent"}}'

# 4. Ver regras iptables específicas do Pod
# Pegar o hash da interface do Pod:
kubectl exec <pod> -- cat /sys/class/net/eth0/iflink
# Encontrar a interface cali<hash> no host correspondente ao ifindex
ip link | grep <ifindex>
# Ver as chains desse endpoint
sudo iptables -L cali-tw-<hash> -n -v
```

**Armadilha comum:** NetworkPolicy com `from.podSelector` sem `namespaceSelector` só permite Pods do **mesmo namespace**. Para permitir o Prometheus do namespace `monitoring` scraping um Pod em `producao`, precisa combinar ambos os seletores.

### calico-node em CrashLoopBackOff

```bash
kubectl logs -n kube-system <calico-node-pod> -c calico-node --previous

# Causas comuns:
# "Failed to create datastore client" → KUBECONFIG ou RBAC errado
# "IP auto-detection failed" → múltiplas interfaces, Calico não sabe qual usar
# "Failed to initialize iptables" → kernel modules br_netfilter/overlay não carregados
# "cali-FORWARD chain missing" → outro agente limpou o iptables

# Para o problema de auto-detection com múltiplas interfaces:
kubectl set env daemonset/calico-node -n kube-system \
  IP_AUTODETECTION_METHOD=interface=eth0
```

### Esgotamento de IPs no IPPool

```bash
# Ver uso atual
calicoctl ipam show
# IPAM stats: 45% allocation

# Ver blocos alocados por node
calicoctl ipam show --show-blocks

# Liberar blocos orfãos (de Pods deletados que não liberaram o IP)
calicoctl ipam check
calicoctl ipam release --ip=<ip-específico>

# Expandir o IPPool (sem downtime — Calico aloca novos blocos do novo range)
# Não é possível reduzir — só expandir
calicoctl patch ippool default-ipv4-ippool \
  --patch='{"spec":{"cidr":"10.244.0.0/14"}}'
```

### BGP Peer derrubado por firewall

```bash
# Verificar se TCP 179 está aberto entre nodes
# No worker1 para worker2:
nc -zv 192.168.3.32 179

# Se bloqueado, adicionar regra no iptables do host (ou no firewall externo):
sudo iptables -I INPUT -p tcp --dport 179 -s 192.168.3.0/24 -j ACCEPT
sudo iptables -I OUTPUT -p tcp --sport 179 -d 192.168.3.0/24 -j ACCEPT
```

---

## Nível Avançado — Edge Cases e Staff

### Felix Configuration — Tuning de Performance

```yaml
apiVersion: projectcalico.org/v3
kind: FelixConfiguration
metadata:
  name: default
spec:
  # Reduz latência de propagação de policy (padrão: 100ms)
  routeRefreshInterval: 30s
  
  # Habilitar packet logging para debugging (caro em produção)
  logSeverityScreen: Warning
  
  # eBPF data plane
  bpfEnabled: false   # true para eBPF
  bpfLogLevel: ""
  
  # Manter conntrack entries por mais tempo (ajuda com long-lived connections)
  netflowMaxOriginalIPsIncluded: 10
  
  # Health check ports (útil para ajustar se tiver conflito)
  healthPort: 9099
  prometheusMetricsPort: 9091
  prometheusMetricsEnabled: true   # expõe métricas do Felix para Prometheus
```

### Prometheus Scraping do Calico

O Felix expõe métricas Prometheus na porta 9091 de cada node. O kube-prometheus-stack já inclui um ServiceMonitor para o calico-node, mas pode precisar de ajuste:

```yaml
# Verificar se o ServiceMonitor existe
kubectl get servicemonitor -n kube-system | grep calico

# Métricas importantes do Felix:
# felix_active_local_endpoints        — número de endpoints locais
# felix_iptables_restore_calls_total  — frequência de iptables-restore
# felix_route_table_list_seconds      — latência de atualização de rotas
# felix_calc_graph_update_time_seconds — tempo de cálculo de policy
```

### BGP Community e Política de Roteamento

Para controlar quais rotas são aceitas/anunciadas (útil em ambientes com múltiplos clusters ou peering com roteadores físicos):

```yaml
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: true    # full mesh (desabilitar ao usar Route Reflectors)
  asNumber: 64512                # AS number do cluster
  serviceClusterIPs:
  - cidr: 10.96.0.0/12           # anunciar ClusterIPs via BGP (acesso externo sem kube-proxy)
  serviceExternalIPs:
  - cidr: 192.168.3.100/32       # anunciar IPs do MetalLB via BGP
```

### Calico eBPF — Substituir kube-proxy

No modo eBPF, o Calico pode substituir completamente o kube-proxy, implementando Services com eBPF (DSR — Direct Server Return, preservação de IP de origem real):

```bash
# 1. Desabilitar kube-proxy
kubectl -n kube-system patch ds kube-proxy \
  -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico":"true"}}}}}'

# 2. Habilitar kube-proxy replacement no Calico
calicoctl patch felixconfiguration default \
  --patch='{"spec":{"bpfKubeProxyIptablesCleanupEnabled":true}}'

kubectl set env daemonset/calico-node -n kube-system \
  FELIX_BPFENABLED=true

# 3. Verificar que Services continuam funcionando
kubectl get svc -A
curl -s http://<clusterip>:<port>
```

### Migração de IPIP para BGP Nativo

Para clusters que começaram com IPIP (comum em instalações iniciais) e querem migrar para roteamento nativo BGP:

```bash
# Verificar se a rede física suporta (nodes no mesmo L2 segment)
ping -c3 192.168.3.32   # worker1 → worker2 diretamente

# Migração gradual — mudar para CrossSubnet primeiro
calicoctl patch ippool default-ipv4-ippool \
  --patch='{"spec":{"ipipMode":"CrossSubnet"}}'

# Verificar que Pods continuam se comunicando
# Se funcionar, desabilitar completamente
calicoctl patch ippool default-ipv4-ippool \
  --patch='{"spec":{"ipipMode":"Never"}}'

# Confirmar: rotas no kernel devem ter "proto bird" sem encapsulation
ip route show | grep bird
```

### Wireshark / tcpdump em Tráfego de Pod

```bash
# Capturar tráfego na interface veth do Pod no host
# 1. Encontrar o ifindex da interface do Pod
kubectl exec <pod> -n <ns> -- cat /sys/class/net/eth0/iflink

# 2. Encontrar a interface no host com esse index
ip link | grep -B1 "^<ifindex>:"
# Retorna: cali<hash>

# 3. Capturar
sudo tcpdump -i cali<hash> -nn -w /tmp/pod-capture.pcap

# 4. Inspecionar (no WSL2, abrir no Windows com Wireshark)
cp /tmp/pod-capture.pcap /mnt/c/Users/felip/Downloads/
```

---

## Referências

- [Calico Documentation](https://docs.tigera.io/calico/latest/about/) — referência completa
- [Calico Architecture](https://docs.tigera.io/calico/latest/reference/architecture/overview) — Felix, BIRD, Typha
- [NetworkPolicy Editor](https://editor.networkpolicy.io/) — visualizador interativo de NetworkPolicy
- [Calico BGP](https://docs.tigera.io/calico/latest/networking/configuring/bgp) — BGP peers, route reflectors
- [Felix Configuration Reference](https://docs.tigera.io/calico/latest/reference/resources/felixconfig) — todas as opções
- [calicoctl Reference](https://docs.tigera.io/calico/latest/reference/calicoctl/) — CLI completa
- DK8S Day5 (Networking) — `/mnt/c/Users/felip/Documents/Kubernets/DK8S`
