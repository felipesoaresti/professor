---
tags:
  - kubernetes
  - services
  - networking
  - kube-proxy
  - metallb
  - dns
area: kubernetes
tipo: conteudo
prerequisites:
  - "[[05-Kubernetes/kubernetes-teoria-inicial]]"
  - "[[05-Kubernetes/statefulset-headless]]"
next:
  - "[[11-Exercicios/services]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Services no Kubernetes

> Conteúdo: [[05-Kubernetes/services]] | Exercícios: [[11-Exercicios/services]] | Trilha: [[00-Trilha/kubernetes]]

Pods nascem e morrem — seus IPs mudam a cada restart. Um Service é uma abstração estável de rede: um IP virtual (ClusterIP) e um nome DNS que permanecem fixos enquanto o Service existe, independente de quantos Pods por trás dele reiniciaram ou se moveram de node. É o mecanismo central de descoberta de serviços no Kubernetes.

---

## O que é e por que existe

Sem Services, para se conectar a um Pod você precisaria saber o IP do Pod — que muda a cada restart. Com Services:

```
App A → ClusterIP do Service B (fixo) → kube-proxy → Pod B-1 ou B-2 ou B-3
```

O Service resolve dois problemas:
1. **Descoberta:** endereço estável (IP + DNS)
2. **Load balancing:** distribui tráfego entre múltiplos Pods com o mesmo selector

### Os 4 tipos de Service

| Tipo | Acessível por | Caso de uso |
|---|---|---|
| **ClusterIP** | Apenas dentro do cluster | Comunicação interna entre serviços |
| **NodePort** | `<node-ip>:<porta>` de fora do cluster | Dev/test, acesso direto a nodes |
| **LoadBalancer** | IP externo provisionado pelo cloud/MetalLB | Exposição produção com IP externo |
| **ExternalName** | DNS — faz CNAME para host externo | Alias para serviços externos ao cluster |

---

## Como funciona internamente

### kube-proxy e as regras de iptables

O `kube-proxy` é um DaemonSet que roda em cada node. Ele assiste à API do Kubernetes e, quando um Service ou Endpoint muda, atualiza as regras de `iptables` (ou `ipvs`) no node.

**Fluxo de uma requisição ClusterIP:**

```
Pod A faz request para 10.96.0.50:80 (ClusterIP do Service)
  → kernel do node intercepta o pacote (iptables DNAT)
  → regra de iptables escolhe aleatoriamente um Pod endpoint (ex: 10.244.2.8:8080)
  → pacote é reescrito (DNAT: destino muda para o IP/porta real do Pod)
  → pacote chega ao Pod B
  → resposta é traduzida de volta (SNAT/MASQUERADE)
```

```bash
# Ver as regras de iptables do Service no node
iptables -t nat -L KUBE-SERVICES -n | grep <cluster-ip>
iptables -t nat -L KUBE-SVC-<hash> -n   # regras de load balancing (round-robin statístico)
```

**Modo IPVS** (alternativo, mais performático para clusters grandes):
- Usa o módulo `ip_vs` do kernel em vez de iptables
- O IPVS tem algoritmos de load balancing reais: round-robin, least-connection, etc.
- Verificar o modo: `kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode`

### Endpoints e EndpointSlices

Quando um Service tem `selector`, o Kubernetes cria automaticamente um objeto `Endpoints` (ou `EndpointSlice` em clusters modernos) que lista os IPs e portas dos Pods que casam com o selector:

```bash
kubectl get endpoints webapp
# NAME     ENDPOINTS                           AGE
# webapp   10.244.1.5:80,10.244.2.8:80        5m

kubectl get endpointslices -l kubernetes.io/service-name=webapp
```

O kube-proxy assiste aos Endpoints e atualiza as regras de iptables quando Pods entram/saem (ficam Ready/NotReady).

> [!info] readinessProbe e Endpoints
> Um Pod só entra nos Endpoints se a readinessProbe passar. Isso é a integração entre [[05-Kubernetes/probes]] e Services: um Pod não-Ready não recebe tráfego automaticamente.

### DNS de Services

O CoreDNS cria registros automáticos para cada Service:

```
<service-name>.<namespace>.svc.cluster.local   → ClusterIP
```

```bash
# De dentro de qualquer Pod no cluster:
nslookup webapp.default.svc.cluster.local
# Address: 10.96.0.50

# Dentro do mesmo namespace, nome curto funciona:
curl http://webapp        # resolve webapp.default.svc.cluster.local

# De outro namespace:
curl http://webapp.default.svc.cluster.local
```

### Campos de porta — port, targetPort, nodePort

```yaml
spec:
  ports:
  - name: http
    port: 80           # porta do Service (acessada pelos clientes)
    targetPort: 8080   # porta no container (onde a app realmente escuta)
    nodePort: 31000    # porta nos nodes (apenas NodePort/LoadBalancer, 30000-32767)
    protocol: TCP
```

`targetPort` pode ser um número ou o nome de um `containerPort`:

```yaml
# No container:
ports:
- name: http
  containerPort: 8080

# No Service:
targetPort: http   # referencia pelo nome — mais flexível
```

---

## Na prática — YAMLs e comandos

### ClusterIP (padrão)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
  namespace: default
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP    # padrão — pode omitir
```

```bash
# Criar imperativo
kubectl expose deployment webapp --port=80 --target-port=8080

# Testar acesso interno (de dentro do cluster)
kubectl run test --image=busybox --restart=Never --rm -it -- \
  wget -qO- http://webapp.default.svc.cluster.local
```

### NodePort

```yaml
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 31080   # se omitir, Kubernetes escolhe entre 30000-32767
```

```bash
# Acessar de fora do cluster (qualquer node)
curl http://192.168.3.31:31080   # worker1
curl http://192.168.3.32:31080   # worker2 — também funciona!
```

> [!warning] NodePort em todos os nodes
> Mesmo que só haja 1 Pod e ele esteja no worker1, o NodePort está aberto em **todos** os nodes. Se o request chega no worker2 mas o Pod está no worker1, o kube-proxy faz um hop extra entre nodes (SNAT). Isso tem implicações de latência e `externalTrafficPolicy`.

### LoadBalancer — com MetalLB no homelab

```yaml
spec:
  type: LoadBalancer
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 8080
```

O MetalLB (Layer 2 mode) aloca um IP do pool configurado e o anuncia via ARP na rede local:

```bash
kubectl get svc webapp
# NAME     TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)
# webapp   LoadBalancer   10.96.0.50    192.168.3.101   80:31080/TCP

# O EXTERNAL-IP é o IP alocado pelo MetalLB — acessível de qualquer máquina na rede
curl http://192.168.3.101
```

**Como o MetalLB L2 funciona:**
1. O MetalLB speaker (DaemonSet) detecta o Service LoadBalancer
2. Um dos nodes é eleito "owner" do IP alocado
3. O speaker no node owner responde a ARP requests para aquele IP
4. O tráfego chega ao node owner → kube-proxy roteia para os Pods

```bash
# Ver o MetalLB em ação
kubectl get pods -n metallb-system
kubectl get ipaddresspools -n metallb-system   # pool de IPs configurado
kubectl get l2advertisements -n metallb-system
```

### ExternalName — alias para serviço externo

```yaml
spec:
  type: ExternalName
  externalName: postgres.staypuff.info   # CNAME para esse host
```

```bash
# Dentro do cluster, qualquer acesso a "db" resolve para o host externo
nslookup db.default.svc.cluster.local
# Alias: postgres.staypuff.info
```

Útil para apontar para um banco de dados externo ao cluster sem hardcodar o host nas aplicações — troca o Service e todos os consumidores seguem.

### Service sem selector — apontar para IP fixo

```yaml
apiVersion: v1
kind: Service
metadata:
  name: legacy-db
spec:
  ports:
  - port: 5432
---
# Endpoints criados manualmente
apiVersion: v1
kind: Endpoints
metadata:
  name: legacy-db   # mesmo nome do Service
subsets:
- addresses:
  - ip: 192.168.3.200   # IP do banco legado externo
  ports:
  - port: 5432
```

O NFS do homelab (`192.168.3.11`) poderia ser exposto assim para ser consumido pelo cluster via DNS interno.

### Multi-port Service

```yaml
spec:
  ports:
  - name: http      # nome obrigatório quando há múltiplas portas
    port: 80
    targetPort: 8080
  - name: metrics
    port: 9090
    targetPort: 9090
  - name: grpc
    port: 9000
    targetPort: 9000
    protocol: TCP
```

### Comandos essenciais

```bash
# Listar Services
kubectl get svc -A
kubectl get svc -n default

# Ver endpoints
kubectl get endpoints webapp
kubectl describe endpoints webapp

# Ver detalhes incluindo selector e qual tipo
kubectl describe svc webapp

# Testar DNS interno
kubectl run dns-test --image=busybox --restart=Never --rm -it -- \
  sh -c "nslookup webapp.default.svc.cluster.local"

# Port-forward para testar sem criar NodePort
kubectl port-forward svc/webapp 8080:80

# Editar Service on-the-fly
kubectl edit svc webapp

# Criar Service a partir de Deployment
kubectl expose deployment webapp --port=80 --target-port=8080 --type=ClusterIP

# Patch para mudar tipo
kubectl patch svc webapp -p '{"spec":{"type":"LoadBalancer"}}'
```

---

## Casos de uso e boas práticas

### Quando usar cada tipo

**ClusterIP:** padrão para comunicação interna. API → banco de dados, frontend → backend. Nunca exposto fora do cluster.

**NodePort:** debug, CI/CD sem MetalLB, acesso temporário. Em produção: evitar — expõe portas em todos os nodes, difícil de gerenciar firewall.

**LoadBalancer:** produção quando você precisa de IP externo. No homelab: usar para o nginx Ingress Controller (que depois roteia por host/path para múltiplos serviços). Não crie LoadBalancer para cada serviço — use 1 LoadBalancer + 1 Ingress.

**ExternalName:** migration pattern — aponta para sistema externo enquanto migra para dentro do cluster.

### Padrão: 1 LoadBalancer + Ingress para múltiplos serviços

```
Internet → 192.168.3.100 (MetalLB LB do nginx Ingress) → nginx Ingress Controller
  → app1.staypuff.info → Service app1 (ClusterIP) → Pods app1
  → app2.staypuff.info → Service app2 (ClusterIP) → Pods app2
```

No homelab, exatamente esse padrão está em uso. O `nginx` Ingress tem 1 LoadBalancer com IP `192.168.3.100`. Todos os outros Services são ClusterIP.

### Session Affinity — sticky sessions

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600   # mesmo cliente → mesmo Pod por 1h
```

> [!warning] Session affinity não é real sticky session
> `ClientIP` usa o IP de origem do cliente. Atrás de um proxy (nginx Ingress), todos os requests chegam com o IP do proxy — a affinity vai concentrar todo o tráfego em um único Pod. Use apenas para casos onde não há proxy na frente.

### `externalTrafficPolicy: Local` — preservar IP do cliente

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local   # padrão: Cluster
```

- **`Cluster` (padrão):** qualquer node pode receber o tráfego e encaminha para qualquer Pod. O IP de origem é perdido (SNAT). Balanceamento uniforme entre Pods.
- **`Local`:** o tráfego só é encaminhado para Pods no **mesmo node** que recebeu o pacote. IP de origem preservado (sem SNAT). Pods em nodes sem réplica não recebem tráfego. Pode gerar desbalanceamento.

Use `Local` quando a aplicação precisa do IP real do cliente (logs, rate limiting por IP). O nginx Ingress Controller usa `Local` para preservar o IP do cliente nos logs.

---

## Troubleshooting — cenários reais de produção

### Cenário 1: Service não roteia — Endpoints vazios

```bash
# Sintoma: curl no ClusterIP retorna "Connection refused" ou timeout
kubectl get endpoints webapp
# NAME     ENDPOINTS   AGE
# webapp   <none>      5m   ← nenhum endpoint!

# Diagnóstico:
# 1. Verificar se o selector do Service casa com os labels dos Pods
kubectl get svc webapp -o jsonpath='{.spec.selector}'
# {app: webapp}

kubectl get pods -l app=webapp
# No resources found.  ← pods não têm o label correto

# 2. Se pods existem, verificar labels
kubectl get pods --show-labels
# webapp-xxx   Running   app=web-app   ← label diferente (web-app vs webapp)

# 3. Corrigir o selector
kubectl patch svc webapp -p '{"spec":{"selector":{"app":"web-app"}}}'

# 4. Verificar se os Pods estão Ready
kubectl get pods -l app=webapp
# 0/1 Running → readinessProbe falhando → não entra nos endpoints
```

### Cenário 2: LoadBalancer com `<pending>` no EXTERNAL-IP

```bash
kubectl get svc webapp
# TYPE: LoadBalancer   EXTERNAL-IP: <pending>   ← MetalLB não alocou IP

# Causas:
# 1. Pool de IPs do MetalLB esgotado
kubectl get ipaddresspools -n metallb-system -o yaml | grep -A5 addresses

# 2. L2Advertisement não configurada para o pool
kubectl get l2advertisements -n metallb-system

# 3. Annotation incorreta do pool
kubectl describe svc webapp | grep -i metallb

# 4. MetalLB speaker não está rodando em algum node
kubectl get pods -n metallb-system -o wide

# Verificar logs do MetalLB
kubectl logs -n metallb-system -l component=speaker --tail=30
```

### Cenário 3: NodePort acessível em um node mas não em outro

```bash
# Sintoma: curl http://192.168.3.31:31080 funciona, mas 192.168.3.32:31080 timeout

# kube-proxy deve estar rodando em todos os nodes
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
# Se não estiver rodando no worker2: o iptables não está atualizado

# Verificar se o iptables tem a regra no node problemático
# (SSH no node)
iptables -t nat -L | grep 31080
# Se não tiver regra → kube-proxy não está sincronizado

# Reiniciar kube-proxy no node (cuidado em produção)
kubectl delete pod <kube-proxy-pod-no-worker2> -n kube-system
```

### Cenário 4: DNS não resolve o Service

```bash
# Dentro de um Pod:
nslookup webapp.default.svc.cluster.local
# nslookup: can't resolve 'webapp.default.svc.cluster.local'

# 1. Verificar se o CoreDNS está rodando
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 2. Verificar a configuração de DNS do Pod
kubectl exec <pod> -- cat /etc/resolv.conf
# nameserver 10.96.0.10   ← IP do Service do CoreDNS
# search default.svc.cluster.local svc.cluster.local cluster.local

# 3. Verificar se o Service do CoreDNS tem Endpoints
kubectl get endpoints kube-dns -n kube-system

# 4. Testar com IP direto (bypassa DNS)
kubectl exec <pod> -- wget -qO- http://10.96.0.50   # ClusterIP direto
```

### Cenário 5: tráfego indo para Pod morto (Endpoints desatualizados)

```bash
# Sintoma: requests ocasionais retornam erro mesmo com Pods "Running"

# Um Pod está Running mas não Ready (readinessProbe falhando)
kubectl get pods -l app=webapp
# webapp-abc   0/1   Running   ← READY=0/1 → não deveria estar nos Endpoints

# Verificar endpoints
kubectl get endpoints webapp
# Verificar se o IP do Pod 0/1 aparece nos Endpoints
# Se aparecer → bug de timing do kube-proxy (raro) ou readinessProbe mal configurada

kubectl describe pod webapp-abc | grep -A10 "Conditions:"
# Ready: False
```

---

## Nível avançado — kube-proxy, IPVS, edge cases

### iptables vs IPVS — a diferença de escala

**iptables mode (padrão):**
- Regras lineares: o kernel verifica cada regra até encontrar a certa
- Em clusters com 10.000 Services → 10.000+ regras → latência de roteamento aumenta
- Cada mudança de Endpoint recria a chain inteira (O(n) updates)

**IPVS mode:**
- Tabela hash: lookup O(1) independente do número de Services
- Algoritmos de LB reais: rr, lc, dh, sh, sed, nq
- Mais performático a partir de ~1000 Services

```bash
# Verificar modo atual
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
# mode: ""   → iptables (padrão)
# mode: ipvs → IPVS

# Em IPVS: ver a tabela virtual
ipvsadm -Ln | grep -A5 <clusterip>
```

### EndpointSlices — escalando além de 1000 Pods

O objeto `Endpoints` tem limite de ~1000 endereços por objeto. Para Deployments grandes, o Kubernetes usa `EndpointSlices` (K8s 1.21+, padrão):

```bash
kubectl get endpointslices -l kubernetes.io/service-name=webapp
# Pode haver múltiplos slices para o mesmo Service
kubectl describe endpointslice webapp-xxxxx
```

### Topology-aware routing — roteamento por zona/node

```yaml
# Service com routing hints por zona (K8s 1.27+)
metadata:
  annotations:
    service.kubernetes.io/topology-mode: auto
```

Com topology-aware routing, o kube-proxy prefere encaminhar para Pods no mesmo node ou zona que o cliente. Reduz latência e tráfego inter-zone em cloud providers.

### Service com múltiplos protocolos (TCP + UDP)

```yaml
spec:
  ports:
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: dns-udp
    port: 53
    protocol: UDP
```

> [!info] Um Service pode misturar TCP e UDP na mesma porta
> Desde K8s 1.20, o `MixedProtocolLBService` feature gate permite LoadBalancer com portas TCP e UDP. O CoreDNS usa exatamente esse padrão.

### Inspecionar MetalLB no homelab

```bash
# Ver o pool de IPs configurado
kubectl get ipaddresspools -n metallb-system -o yaml
# spec.addresses: [192.168.3.100-192.168.3.110]

# Ver qual IP cada Service está usando
kubectl get svc -A | grep LoadBalancer

# Ver qual node é o "speaker" atual para um IP
# (olhar os logs do DaemonSet speaker)
kubectl logs -n metallb-system -l component=speaker | grep "acquired"

# O Ingress nginx usa o primeiro IP do pool:
kubectl get svc -n ingress-nginx
# EXTERNAL-IP: 192.168.3.100
```

### Service Account e NetworkPolicy — restringindo acesso ao Service

Com Calico (CNI do homelab), é possível criar NetworkPolicy que restringe quais Pods podem acessar um Service:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: webapp-access
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: webapp
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend   # apenas Pods com label role=frontend
    ports:
    - port: 8080
```

---

## Referências

- **Kubernetes docs** — [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- **Kubernetes docs** — [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- **MetalLB docs** — metallb.universe.tf (L2 mode usado no homelab)
- **kube-proxy iptables** — `iptables -t nat -L` no node para ver regras geradas
- **IPVS deep dive** — kubernetes.io/blog/2018/07/09/ipvs-based-in-cluster-load-balancing-deep-dive/
- **Linuxtips DK8S** — `/mnt/c/Users/felip/Documents/Kubernets/DK8S`
- `kubectl explain service.spec` — referência completa da API
