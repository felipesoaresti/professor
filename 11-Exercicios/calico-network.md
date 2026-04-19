---
tags:
  - exercicios
  - networking
  - kubernetes
  - calico
  - networkpolicy
tipo: exercicios
area: networking
conteudo: "[[02-Networking/calico-network]]"
trilha: "[[00-Trilha/networking]]"
---

# Exercícios — Calico Network

> Conteúdo: [[02-Networking/calico-network]] | Trilha: [[00-Trilha/networking]]

> [!warning] Restrições
> Namespaces `controle-gastos`, `promobot` e `databases` → **somente leitura**.
> Criação de NetworkPolicies nos namespaces de produção está **proibida** — NetworkPolicy mal configurada pode quebrar aplicações em produção.
> Todos os exercícios de criação usam o namespace `estudo`.

---

## Exercício 1: Anatomia do Calico no Cluster

**Contexto:** Você assumiu a gestão de um cluster que tem Calico instalado. Nunca ninguém documentou como ele está configurado. Antes de fazer qualquer alteração, você precisa entender o estado atual.

**Missão:** Fazer um inventário completo da instalação Calico — componentes, configuração de rede e estado BGP.

**Requisitos:**
- [ ] Listar todos os Pods do Calico e verificar que estão `Running` (namespace `kube-system` e `calico-system`)
- [ ] Identificar se o cluster usa Tigera Operator ou manifesto direto (verificar se existe CRD `installations.operator.tigera.io`)
- [ ] Inspecionar o IPPool: CIDR, blockSize, ipipMode, vxlanMode, natOutgoing
- [ ] Verificar o estado BGP de **cada node**: quantas sessões BGP estão `Established` e quais peers cada node tem
- [ ] No k8s-master, inspecionar a tabela de roteamento e identificar as rotas instaladas pelo BIRD (`proto bird`): qual CIDR vai para qual node
- [ ] Listar as FelixConfigurations existentes e identificar se o modo eBPF está habilitado

**Verificação:**
```bash
# Componentes
sudo kubectl get pods -n kube-system | grep calico
sudo kubectl get pods -n calico-system 2>/dev/null || echo "sem calico-system"

# IPPool
sudo calicoctl get ippool -o yaml

# Estado BGP (rodar em cada node)
sudo calicoctl node status

# Rotas no kernel
ip route show | grep bird
```

---

## Exercício 2: Rastrear o Caminho de um Pacote entre Pods

**Contexto:** Um desenvolvedor reporta "comunicação lenta entre microserviços em nodes diferentes". Antes de investigar a aplicação, você quer entender exatamente o caminho que um pacote percorre e confirmar que não há overhead de encapsulation.

**Missão:** Criar dois Pods em nodes diferentes e rastrear o caminho completo do pacote — do Pod A até o Pod B.

**Requisitos:**
- [ ] Criar dois Pods no namespace `estudo` — um no `k8s-worker1` e outro no `k8s-worker2` (usar `nodeSelector`)
- [ ] Confirmar os IPs alocados e em qual bloco `/26` cada IP se enquadra
- [ ] A partir do Pod no worker1, fazer `ping` para o Pod no worker2 e capturar o tráfego na **interface do host** (não dentro do Pod) usando `tcpdump`
- [ ] Verificar no tcpdump se o pacote tem encapsulation (IPIP ou VXLAN) ou é roteamento direto L3
- [ ] Usar `ip route get <ip-do-pod-destino>` no worker1 para confirmar a rota e o next-hop usado
- [ ] Identificar qual interface `cali<hash>` no worker2 corresponde ao Pod destino

**Verificação:**
```bash
# IPs dos Pods
sudo kubectl get pods -n estudo -o wide

# Rota para o Pod destino (rodar no worker1)
ip route get <ip-do-pod-no-worker2>

# Capture na interface física do worker1 durante o ping
# Pacote sem encapsulation: src=<ip-pod-a> dst=<ip-pod-b> (IPs de Pod no wire)
# Pacote com IPIP: src=192.168.3.31 dst=192.168.3.32 com payload IP interno
sudo tcpdump -i eth0 -nn host <ip-do-pod-b>
```

---

## Exercício 3: NetworkPolicy — Default Deny e Allowlist

**Contexto:** O namespace `estudo` atualmente não tem nenhuma NetworkPolicy. Qualquer Pod pode falar com qualquer outro. O time de segurança exigiu isolamento: tráfego bloqueado por padrão, com allowlist explícita.

**Missão:** Implementar o modelo de segurança "default deny + allowlist" no namespace `estudo` e verificar que funciona corretamente.

**Requisitos:**
- [ ] Criar três Pods no namespace `estudo`: `frontend`, `backend`, e `intruso`
- [ ] Antes de criar qualquer NetworkPolicy, confirmar que todos se comunicam livremente
- [ ] Criar uma NetworkPolicy `default-deny` que bloqueie **todo** ingress e egress de todos os Pods no namespace
- [ ] Confirmar que o `frontend` não consegue mais pingar o `backend`
- [ ] Criar NetworkPolicies adicionais que:
  - Permitem egress de DNS (UDP/TCP porta 53) para todos os Pods
  - Permitem o `frontend` fazer requests para o `backend` na porta 80
  - O Pod `intruso` **não deve** conseguir alcançar o `backend` em nenhum momento
- [ ] Testar e documentar: o que acontece quando o `frontend` tenta acessar a internet (ex: `curl google.com`)? Por quê?

**Verificação:**
```bash
# Teste de conectividade (deve falhar após default-deny, funcionar após allowlist)
sudo kubectl exec -n estudo frontend -- wget --timeout=3 -qO- http://backend
sudo kubectl exec -n estudo intruso -- wget --timeout=3 -qO- http://backend

# Ver NetworkPolicies aplicadas
sudo kubectl get networkpolicy -n estudo
sudo kubectl describe networkpolicy -n estudo
```

---

## Exercício 4: Debugging de NetworkPolicy — "Broken by Policy"

**Contexto:** Um colega aplicou NetworkPolicies no namespace `estudo` e "quebrou tudo". Ele foi embora e deixou você para resolver. Os Pods estão funcionando mas não se comunicam.

**Missão:** Diagnosticar qual(is) NetworkPolicy(ies) está(ão) bloqueando o tráfego e corrigi-las sem deletar tudo e recomeçar.

**Requisitos:**
- [ ] Criar o cenário com os YAMLs abaixo (salvar como `broken-policies.yaml` e aplicar):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
  namespace: estudo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-a
  template:
    metadata:
      labels:
        app: app-a
        tier: frontend
    spec:
      containers:
      - name: app
        image: nginx:alpine
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-b
  namespace: estudo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-b
  template:
    metadata:
      labels:
        app: app-b
        tier: backend
    spec:
      containers:
      - name: app
        image: nginx:alpine
---
apiVersion: v1
kind: Service
metadata:
  name: app-b-svc
  namespace: estudo
spec:
  selector:
    app: app-b
  ports:
  - port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: policy-1
  namespace: estudo
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: frontend
    - podSelector:
        matchLabels:
          tier: frontend
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: policy-2
  namespace: estudo
spec:
  podSelector:
    matchLabels:
      app: app-a
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: app-b
          tier: backend
    ports:
    - port: 8080
```

- [ ] Tentar acessar `app-b` a partir de `app-a` e confirmar que falha
- [ ] Usando **apenas `kubectl describe`** e leitura dos YAMLs, identificar **dois problemas** nessas policies sem usar trial-and-error
- [ ] Corrigir os problemas e confirmar que `app-a` consegue acessar `app-b`

**Verificação:**
```bash
sudo kubectl exec -n estudo deploy/app-a -- wget --timeout=5 -qO- http://app-b-svc
# Deve retornar o HTML do nginx após correção
```

> [!tip] Dica de Análise
> Examine os seletores com cuidado. NetworkPolicy com múltiplos seletores em `from` tem semântica AND vs OR dependendo de como estão estruturados. E verifique as portas nos dois lados.

---

## Exercício 5: Calico GlobalNetworkPolicy — Regra de Baseline

**Contexto:** O time de segurança quer uma regra **cluster-wide** que bloqueie acesso ao metadata service da cloud (169.254.169.254) de todos os Pods — proteção contra SSRF. Além disso, quer garantir que o Prometheus consiga scraping em qualquer namespace.

**Missão:** Criar GlobalNetworkPolicies de baseline usando os CRDs do Calico.

**Requisitos:**
- [ ] Verificar que o `calicoctl` está instalado e funcionando (`calicoctl version`)
- [ ] Criar uma `GlobalNetworkPolicy` chamada `bloquear-metadata` com:
  - `order: 5` (alta prioridade)
  - Aplicada a todos os endpoints (`selector: all()`)
  - Bloqueia egress para `169.254.169.254/32`
  - Permite todo o resto (regra Allow no final)
- [ ] Criar uma `GlobalNetworkPolicy` chamada `permitir-prometheus` com:
  - `order: 10`
  - Aplicada a Pods com label `prometheus.io/scrape: "true"`
  - Permite ingress da porta 9090, 9091, 8080, e 8443 a partir de Pods com label `app.kubernetes.io/name: prometheus`
- [ ] Testar a policy de metadata: de um Pod no namespace `estudo`, tentar `curl --connect-timeout 3 http://169.254.169.254` — deve dar timeout

**Verificação:**
```bash
# Ver GlobalNetworkPolicies
sudo calicoctl get globalnetworkpolicy -o yaml

# Testar bloqueio (deve dar timeout/connection refused)
sudo kubectl exec -n estudo <pod> -- curl --connect-timeout 3 http://169.254.169.254
echo "Exit code: $?"   # deve ser diferente de 0
```

> [!warning] GlobalNetworkPolicy afeta o cluster inteiro
> Teste com `calicoctl apply --dry-run` primeiro se disponível na versão instalada.
> Aplique em horário de baixo tráfego e tenha rollback pronto (`calicoctl delete globalnetworkpolicy <nome>`).

---

## Exercício 6: NetworkSet — Agrupar CIDRs Externos

**Contexto:** A aplicação no namespace `estudo` só deve acessar servidores externos conhecidos: o NFS do homelab (192.168.3.11) e o DNS externo (8.8.8.8). Todo outro egress externo deve ser bloqueado. Ao invés de hardcodar IPs em cada NetworkPolicy, usar NetworkSet para reutilização.

**Missão:** Criar um GlobalNetworkSet com os CIDRs autorizados e uma NetworkPolicy que usa esse set.

**Requisitos:**
- [ ] Criar um `GlobalNetworkSet` chamado `destinos-autorizados` com as nets:
  - `192.168.3.11/32` (NFS)
  - `8.8.8.8/32` e `8.8.4.4/32` (DNS externo)
  - `192.168.3.0/24` (rede do homelab — acesso ao cluster em si)
  - Label: `role: authorized-external`
- [ ] Criar uma NetworkPolicy no namespace `estudo` que:
  - Aplica em todos os Pods (`podSelector: {}`)
  - Bloqueia todo egress por padrão
  - Permite egress para Pods dentro do namespace (`podSelector: {}`)
  - Permite egress para o `GlobalNetworkSet` com label `role: authorized-external`
  - Permite egress de DNS (UDP 53)
- [ ] Verificar: a partir de um Pod, `curl http://192.168.3.11` deve funcionar mas `curl http://1.1.1.1` deve falhar

**Verificação:**
```bash
sudo calicoctl get globalnetworkset -o yaml

# Deve funcionar (NFS está no NetworkSet)
sudo kubectl exec -n estudo <pod> -- curl --connect-timeout 3 http://192.168.3.11

# Deve falhar (1.1.1.1 não está no NetworkSet)
sudo kubectl exec -n estudo <pod> -- curl --connect-timeout 3 http://1.1.1.1
```

---

## Exercício 7: IPAM — Inspecionar e Diagnosticar

**Contexto:** O time de infra reportou que um node "ficou sem IPs para Pods". Você precisa entender o estado do IPAM, encontrar a causa, e saber como liberar IPs orfãos (de Pods já deletados que não liberaram o IP corretamente).

**Missão:** Fazer análise completa do IPAM do Calico e simular a resolução de IP leak.

**Requisitos:**
- [ ] Verificar o uso total de IPs no cluster (`calicoctl ipam show`)
- [ ] Ver a distribuição de blocos por node (`calicoctl ipam show --show-blocks`) — identificar quantos IPs cada node tem alocados vs disponíveis
- [ ] Rodar `calicoctl ipam check` para identificar inconsistências entre o IPAM e os Pods reais do cluster
- [ ] Inspecionar um IP específico de um Pod em execução: `calicoctl ipam show --ip=<ip-de-um-pod>` — identificar para qual Pod/node ele está alocado
- [ ] Calcular: com o blockSize atual e o CIDR do IPPool, qual é a capacidade máxima de Pods por node? E do cluster inteiro?
- [ ] Responder: se um node for adicionado, ele recebe um bloco automaticamente? Quando?

**Verificação:**
```bash
sudo calicoctl ipam show
sudo calicoctl ipam show --show-blocks
sudo calicoctl ipam check
sudo calicoctl ipam show --ip=<ip-de-pod-existente>
```

---

## Exercício 8: Simular Falha de BGP e Diagnosticar (Staff-level)

**Contexto:** Às 3h da manhã você recebe alerta: "Pods no worker2 não conseguem alcançar Pods no worker1". O cluster está parcialmente operacional. Você precisa diagnosticar e restaurar a comunicação.

**Missão:** Simular a falha de BGP em um node, diagnosticar como o Calico detecta e tenta se recuperar, e restaurar o estado.

**Requisitos:**

**Parte A — Setup:**
- [ ] Criar dois Pods no namespace `estudo` — um em cada worker — e confirmar que se comunicam (ping bem-sucedido)
- [ ] Verificar que as sessões BGP estão `Established` em ambos os workers

**Parte B — Simular a falha:**
- [ ] No **worker1**, bloquear a porta BGP (TCP 179) usando iptables para simular firewall entre nodes:
  ```bash
  sudo iptables -I INPUT -p tcp --dport 179 -j DROP
  sudo iptables -I OUTPUT -p tcp --sport 179 -j DROP
  ```
- [ ] Aguardar ~90 segundos (hold timer BGP padrão) e verificar o estado BGP no worker1
- [ ] Verificar que a rota para Pods do worker1 desapareceu da tabela de roteamento do worker2
- [ ] Tentar pingar do Pod no worker2 para o Pod no worker1 — deve falhar

**Parte C — Diagnóstico:**
- [ ] Sem a informação de que o iptables foi alterado, diagnosticar a causa usando:
  - `calicoctl node status` no worker1
  - `ip route show` no worker2
  - Logs do calico-node no worker1
- [ ] Documentar quais comandos você rodaria e em que ordem para identificar que é problema de firewall bloqueando BGP

**Parte D — Restauração:**
- [ ] Remover as regras iptables do worker1
- [ ] Verificar que o BGP se restabelece automaticamente (pode levar até 30s)
- [ ] Confirmar que as rotas voltam e o ping entre Pods funciona novamente

**Verificação:**
```bash
# Estado BGP — deve mostrar peer em "Idle" ou "Active" durante a falha
sudo calicoctl node status

# Rotas no worker2 — rota para bloco do worker1 deve desaparecer
ip route show | grep bird

# Após restauração
sudo kubectl exec -n estudo <pod-worker2> -- ping -c3 <ip-pod-worker1>
```

> [!warning] Cleanup Obrigatório
> Remover as regras iptables ao finalizar:
> ```bash
> sudo iptables -D INPUT -p tcp --dport 179 -j DROP
> sudo iptables -D OUTPUT -p tcp --sport 179 -j DROP
> ```
> Verificar que não sobraram regras: `sudo iptables -L -n | grep 179`
