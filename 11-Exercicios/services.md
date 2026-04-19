---
tags:
  - exercicios
  - kubernetes
  - services
  - networking
  - kube-proxy
  - metallb
tipo: exercicios
area: kubernetes
conteudo: "[[05-Kubernetes/services]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Exercícios — Services no Kubernetes

> Conteúdo: [[05-Kubernetes/services]] | Trilha: [[00-Trilha/kubernetes]]

**Infraestrutura:** k8s-master (192.168.3.30), worker1 (.31), worker2 (.32)
**Namespace de trabalho:** `estudo` (criar para estes exercícios)
**Atenção:** Nunca tocar em `controle-gastos`, `promobot`, `databases/postgres`.

---

## Exercício 1: Inventário de Services no cluster (leitura apenas)

**Contexto:** Você é novo no time e precisa entender quais Services existem, de quais tipos, e como estão distribuídos pelos namespaces — antes de criar qualquer coisa.

**Missão:** Faça um inventário completo dos Services do cluster sem alterar nada.

**Requisitos:**
- [ ] Listar todos os Services de todos os namespaces: `kubectl get svc -A`
- [ ] Identificar quantos Services de cada tipo existem (`ClusterIP`, `NodePort`, `LoadBalancer`)
- [ ] Para os Services do tipo `LoadBalancer`: registrar qual IP foi atribuído pelo MetalLB
- [ ] Verificar os Endpoints de cada Service no namespace `databases`:
  - `kubectl get endpoints -n databases`
  - Quantos IPs/portas estão registrados?
- [ ] Verificar o Service do nginx Ingress Controller (`ingress-nginx` ou similar): qual tipo é? Qual porta?
- [ ] Para um Service ClusterIP qualquer: verificar qual `targetPort` está configurado vs a porta do container (`kubectl describe svc <nome> -n <ns>`)
- [ ] Documentar: qual Service tem `CLUSTER-IP: None`? O que isso indica?

**Verificação:**
```bash
kubectl get svc -A --no-headers | awk '{print $4}' | sort | uniq -c | sort -rn
# Conta Services por tipo (ClusterIP, NodePort, LoadBalancer, None)

kubectl get svc -A | grep LoadBalancer
# Deve mostrar o IP do MetalLB na coluna EXTERNAL-IP (192.168.3.x)

kubectl get endpoints -n databases
# Deve mostrar os IPs dos Pods do StatefulSet postgres
```

---

## Exercício 2: ClusterIP e resolução DNS interna

**Contexto:** O time precisa de uma aplicação de teste para demonstrar como Services ClusterIP funcionam internamente — incluindo DNS, balanceamento e isolamento de namespace.

**Missão:** Crie um Deployment + Service ClusterIP e demonstre o funcionamento completo do DNS interno.

**Requisitos:**
- [ ] Criar namespace `estudo`
- [ ] Criar Deployment `nginx-app` com 3 réplicas usando `nginx:1.27-alpine`
- [ ] Criar Service ClusterIP `nginx-svc` apontando para os Pods do Deployment (porta 80)
- [ ] Verificar que os 3 Pods aparecem nos Endpoints: `kubectl get endpoints nginx-svc -n estudo`
- [ ] Criar Pod temporário de teste: `kubectl run dns-test -n estudo --image=busybox:1.36 --restart=Never -- sleep 3600`
- [ ] Dentro do Pod, testar as formas de resolução DNS:
  - `nslookup nginx-svc` (nome curto — funciona dentro do mesmo namespace?)
  - `nslookup nginx-svc.estudo` (nome com namespace)
  - `nslookup nginx-svc.estudo.svc.cluster.local` (FQDN completo)
- [ ] Verificar que o IP retornado é o `CLUSTER-IP` do Service
- [ ] Testar conectividade: `wget -qO- nginx-svc.estudo.svc.cluster.local`
- [ ] Deletar o Pod de teste ao final (manter Deployment e Service para próximos exercícios)

**Verificação:**
```bash
kubectl get svc nginx-svc -n estudo
# CLUSTER-IP deve ser um IP do range 10.x.x.x

kubectl get endpoints nginx-svc -n estudo
# Deve listar 3 IPs (um por Pod)

kubectl exec -n estudo dns-test -- nslookup nginx-svc.estudo.svc.cluster.local
# Deve retornar o ClusterIP do Service
```

---

## Exercício 3: NodePort — exposição externa básica

**Contexto:** O time quer acessar a aplicação de fora do cluster durante o desenvolvimento, antes de configurar um Ingress. NodePort é a solução mais simples para isso.

**Missão:** Converta o Service para NodePort e acesse a aplicação de fora do cluster.

**Requisitos:**
- [ ] Criar Service NodePort `nginx-nodeport` no namespace `estudo`:
  - `port: 80` (porta do Service)
  - `targetPort: 80` (porta do container)
  - `nodePort: 30080` (porta exposta nos nodes — fixar para facilitar o teste)
- [ ] Verificar que o NodePort foi registrado: `kubectl get svc nginx-nodeport -n estudo`
- [ ] Testar o acesso de fora do cluster (do k8s-master ou de outra máquina):
  - `curl http://192.168.3.31:30080` (worker1)
  - `curl http://192.168.3.32:30080` (worker2)
  - `curl http://192.168.3.30:30080` (master)
- [ ] Confirmar que os 3 endereços funcionam — mesmo que o Pod esteja apenas em um node
- [ ] Observar o header `Server` na resposta para confirmar que é o nginx
- [ ] Verificar as regras iptables geradas para o NodePort:
  ```bash
  sudo iptables -t nat -L -n | grep 30080
  ```
- [ ] Deletar apenas o Service NodePort ao final (manter o Deployment)

**Verificação:**
```bash
kubectl get svc nginx-nodeport -n estudo
# EXTERNAL-IP: <none>  PORT(S): 80:30080/TCP

curl -s http://192.168.3.31:30080 | grep -i nginx
# Deve retornar a página de boas-vindas do nginx

curl -s http://192.168.3.30:30080 | grep -i nginx
# Também funciona via master (kube-proxy no master também aceita NodePort)
```

---

## Exercício 4: LoadBalancer com MetalLB

**Contexto:** A aplicação vai para staging e precisa de um IP dedicado acessível na rede local — sem depender de NodePort. O MetalLB está configurado com um pool de IPs disponíveis.

**Missão:** Crie um Service LoadBalancer e observe a alocação de IP pelo MetalLB.

**Requisitos:**
- [ ] Criar Service LoadBalancer `nginx-lb` no namespace `estudo` (porta 80)
- [ ] Monitorar a alocação do IP: `kubectl get svc nginx-lb -n estudo -w`
- [ ] Registrar qual IP foi atribuído (deve ser do range 192.168.3.x, diferente de .100)
- [ ] Verificar que o IP é acessível da sua máquina (WSL/Windows): `curl http://<IP-alocado>`
- [ ] Observar o objeto que o MetalLB criou para anunciar o IP:
  ```bash
  kubectl get ipaddresspool -A
  kubectl get l2advertisement -A
  ```
- [ ] Inspecionar o evento de alocação: `kubectl describe svc nginx-lb -n estudo | grep -A5 Events`
- [ ] Escalar o Deployment para 0 réplicas e verificar que o IP ainda está alocado (Service existe, Pods não)
- [ ] Escalar de volta para 3 e confirmar que o IP continua o mesmo
- [ ] Deletar o Service e confirmar que o IP foi liberado (verifique com `kubectl get svc -n estudo`)

**Verificação:**
```bash
kubectl get svc nginx-lb -n estudo
# EXTERNAL-IP: 192.168.3.x (não .100, que é o nginx ingress)

curl -s http://<IP-alocado> | grep -i nginx
# Resposta do nginx

kubectl scale deployment nginx-app --replicas=0 -n estudo
kubectl get svc nginx-lb -n estudo
# EXTERNAL-IP permanece mesmo sem Pods
```

---

## Exercício 5: Service sem selector — Endpoints manual

**Contexto:** A aplicação precisa acessar um banco de dados legado que roda fora do Kubernetes (em 192.168.3.11, porta 5432). Você precisa criar um Service sem selector e registrar o Endpoint manualmente para que a aplicação use DNS interno normalmente.

**Missão:** Configure um Service sem selector que aponte para um IP externo ao cluster.

**Requisitos:**
- [ ] Criar Service `db-externo` no namespace `estudo` **sem** campo `selector`:
  - `port: 5432`
  - `protocol: TCP`
  - tipo: `ClusterIP`
- [ ] Verificar que os Endpoints estão vazios: `kubectl get endpoints db-externo -n estudo`
- [ ] Criar objeto Endpoints (mesmo nome que o Service) apontando para `192.168.3.11:5432`:
  ```yaml
  apiVersion: v1
  kind: Endpoints
  metadata:
    name: db-externo
    namespace: estudo
  subsets:
    - addresses:
        - ip: 192.168.3.11
      ports:
        - port: 5432
  ```
- [ ] Verificar que os Endpoints aparecem agora: `kubectl get endpoints db-externo -n estudo`
- [ ] Criar Pod de teste e resolver o DNS:
  - `nslookup db-externo.estudo.svc.cluster.local` → deve retornar o ClusterIP do Service
  - `curl telnet db-externo.estudo.svc.cluster.local:5432` (pode dar erro de protocolo — o importante é o TCP chegar)
- [ ] Documentar: quando um Pod resolve `db-externo.estudo.svc.cluster.local`, para qual IP o tráfego é DNAT-ado?
- [ ] Deletar os recursos ao final

**Verificação:**
```bash
kubectl get endpoints db-externo -n estudo
# Deve mostrar: 192.168.3.11:5432

kubectl exec -n estudo dns-test -- nslookup db-externo.estudo.svc.cluster.local
# Retorna o ClusterIP do Service (não o 192.168.3.11 diretamente)
# O kube-proxy cuida do DNAT para 192.168.3.11:5432
```

---

## Exercício 6: externalTrafficPolicy — Local vs Cluster

**Contexto:** O time de segurança exige que o IP real do cliente seja preservado nos logs do nginx. A política padrão (`Cluster`) aplica SNAT e perde o IP original. Você vai demonstrar a diferença entre `Local` e `Cluster` na prática.

**Missão:** Configure e compare os dois modos de `externalTrafficPolicy` em um Service LoadBalancer.

**Requisitos:**
- [ ] Recriar o Service `nginx-lb` com `externalTrafficPolicy: Cluster` (padrão)
- [ ] Fazer requisições e verificar o IP registrado nos logs do nginx:
  ```bash
  kubectl logs -l app=nginx-app -n estudo --tail=5
  ```
  - O IP nos logs é o IP do cliente ou um IP do node?
- [ ] Mudar o Service para `externalTrafficPolicy: Local`:
  ```bash
  kubectl patch svc nginx-lb -n estudo -p '{"spec":{"externalTrafficPolicy":"Local"}}'
  ```
- [ ] Verificar nos logs novamente — o IP do cliente agora aparece?
- [ ] **Cenário de falha:** Identificar em qual node cada Pod está rodando (`kubectl get pods -n estudo -o wide`)
- [ ] Testar via NodePort diretamente em um node **sem** Pod nginx — a requisição funciona com `Local`?
- [ ] Documentar: por que `externalTrafficPolicy: Local` pode causar balanceamento desigual?
- [ ] Voltar para `Cluster` ao final

**Verificação:**
```bash
# Com externalTrafficPolicy: Cluster
kubectl logs -l app=nginx-app -n estudo --tail=3
# IP nos logs: 10.x.x.x (IP interno do node — SNAT aplicado)

# Com externalTrafficPolicy: Local
kubectl logs -l app=nginx-app -n estudo --tail=3
# IP nos logs: 192.168.3.x (IP real do cliente)

# Node sem Pod com policy=Local:
curl http://<IP-do-node-sem-pod>:30080
# Connection refused ou timeout — o tráfego não é redirecionado para outro node
```

---

## Exercício 7: Diagnosticar Endpoints vazios

**Contexto:** Um desenvolvedor criou um Service mas a aplicação não está acessível. Ele diz que "o Service existe e os Pods estão rodando". Você precisa diagnosticar sistematicamente sem acesso à aplicação.

**Missão:** Execute o fluxo completo de diagnóstico de Service com Endpoints vazios.

**Requisitos:**
- [ ] Criar Service `app-quebrado` no namespace `estudo` com selector `app: minha-app`
- [ ] Criar Deployment `minha-app` com label `app: meu-app` (typo intencional — note o singular)
- [ ] Verificar o sintoma: `kubectl get endpoints app-quebrado -n estudo` — Endpoints vazios?
- [ ] Executar o diagnóstico completo:
  1. `kubectl describe svc app-quebrado -n estudo` → ver o selector configurado
  2. `kubectl get pods -n estudo --show-labels` → ver os labels reais dos Pods
  3. Comparar: selector do Service vs labels dos Pods
  4. `kubectl get pods -n estudo -l app=minha-app` → deve retornar nenhum Pod
  5. `kubectl get pods -n estudo -l app=meu-app` → deve retornar os Pods
- [ ] Corrigir o Deployment para usar o label correto (`app: minha-app`) OU corrigir o Service
- [ ] Confirmar que os Endpoints aparecem após a correção
- [ ] Testar conectividade: `curl http://app-quebrado.estudo.svc.cluster.local` de um Pod de teste
- [ ] Deletar os recursos ao final

**Verificação:**
```bash
kubectl get endpoints app-quebrado -n estudo
# Antes da correção: <none>
# Após a correção: 10.x.x.x:80,10.x.x.x:80,...

kubectl describe svc app-quebrado -n estudo | grep Selector
# Selector: app=minha-app

kubectl get pods -n estudo --show-labels | grep meu-app
# O label real é "meu-app", não "minha-app" — daí o mismatch
```

---

## Exercício 8: Rastrear o fluxo iptables do kube-proxy (Staff-level)

**Contexto:** Um incidente de produção revelou que requisições para um Service estavam indo para um Pod que já tinha sido deletado. Você precisa entender exatamente como o kube-proxy programa as regras iptables e como diagnosticar atrasos na propagação de Endpoints.

**Missão:** Rastreie manualmente o caminho completo de uma requisição para um ClusterIP através das regras iptables no node.

**Requisitos:**
- [ ] Garantir que o Deployment `nginx-app` está rodando com 3 réplicas no namespace `estudo`
- [ ] No k8s-master, obter o ClusterIP do Service `nginx-svc`:
  ```bash
  sudo kubectl get svc nginx-svc -n estudo -o jsonpath='{.spec.clusterIP}'
  ```
- [ ] Inspecionar a chain de DNAT criada pelo kube-proxy:
  ```bash
  sudo iptables -t nat -L KUBE-SERVICES -n | grep <CLUSTER-IP>
  # Deve mostrar um salto para KUBE-SVC-<hash>
  
  sudo iptables -t nat -L KUBE-SVC-<hash> -n
  # Deve mostrar as regras de balanceamento com probabilidade (1/3, 1/2, 1/1)
  
  sudo iptables -t nat -L KUBE-SEP-<hash> -n
  # Deve mostrar a regra DNAT para o IP real do Pod
  ```
- [ ] Registrar os IPs de destino de cada `KUBE-SEP-*` e comparar com `kubectl get endpoints nginx-svc -n estudo`
- [ ] **Simular propagação lenta:** deletar um Pod e cronometrar quanto tempo leva para o iptables remover a entrada correspondente
  ```bash
  kubectl delete pod <nome-do-pod> -n estudo
  # Em outro terminal: watch sudo iptables -t nat -L KUBE-SVC-<hash> -n
  ```
- [ ] Após a remoção, confirmar que o Pod deletado não aparece mais nas regras
- [ ] Documentar: qual componente é responsável por atualizar as regras iptables quando um Pod muda? Qual é a latência típica?
- [ ] Limpeza final do namespace `estudo`:
  ```bash
  kubectl delete namespace estudo
  ```

**Verificação:**
```bash
sudo iptables -t nat -L KUBE-SERVICES -n | grep <CLUSTER-IP>
# KUBE-SVC-<hash> → entry para o ClusterIP

sudo iptables -t nat -L KUBE-SVC-<hash> -n
# 3 regras com statistic --probability 0.33333, 0.50000, 1.00000
# (probabilidade cumulativa para distribuir 1/3 cada)

# Os IPs em KUBE-SEP-* devem bater com:
kubectl get endpoints nginx-svc -n estudo
# 10.x.x.x:80, 10.y.y.y:80, 10.z.z.z:80
```

---

> [!tip] Ordem recomendada
> Exercício 1 é leitura pura — sem risco, sempre primeiro.
> Exercícios 2, 3 e 4 são sequenciais e progressivos — cada tipo de Service sobre o mesmo Deployment.
> Exercício 5 é independente — pode ser feito a qualquer momento após criar o namespace.
> Exercício 6 requer LoadBalancer — faça após o exercício 4.
> Exercício 7 é o mais importante para o dia a dia — diagnóstico de Endpoints vazios é o problema #1 com Services.
> Exercício 8 é avançado — requer acesso SSH ao k8s-master e familiaridade com iptables.

> [!warning] MetalLB e IPs
> O MetalLB aloca IPs sequencialmente do pool configurado. O IP 192.168.3.100 já está em uso pelo nginx Ingress Controller. Ao criar Services LoadBalancer, verifique que o IP alocado é diferente de .100 para evitar conflito.

> [!info] Limpeza
> Ao final dos exercícios, deletar o namespace `estudo` remove todos os recursos criados (Deployments, Services, Pods). Os IPs do MetalLB são automaticamente liberados quando os Services LoadBalancer são deletados.
