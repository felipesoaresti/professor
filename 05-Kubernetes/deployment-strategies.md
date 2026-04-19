---
tags:
  - kubernetes
  - deployment
  - strategies
  - rolling-update
  - canary
  - blue-green
area: kubernetes
tipo: conteudo
prerequisites:
  - "[[05-Kubernetes/kubernetes-teoria-inicial]]"
next:
  - "[[11-Exercicios/deployment-strategies]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Deployment Strategies

> Conteúdo: [[05-Kubernetes/deployment-strategies]] | Exercícios: [[11-Exercicios/deployment-strategies]] | Trilha: [[00-Trilha/kubernetes]]

Toda atualização de software envolve uma troca entre risco e velocidade. Deployment strategies são os mecanismos que controlam *como* novos Pods substituem os antigos — determinando quanto downtime ocorre, quanto risco é exposto aos usuários, e quão rápido a troca acontece.

---

## O que é e por que existe

Antes de entender as strategies, é preciso entender o que acontece quando você faz `kubectl apply` em um Deployment com nova imagem:

1. O Deployment Controller detecta mudança no `spec.template`
2. Cria um novo **ReplicaSet** com a nova versão
3. A **strategy** define como o RS antigo é escalonado para zero e o novo RS cresce

Sem controle de strategy, você teria duas opções ruins:
- Matar tudo e recriar → **downtime garantido**
- Deixar conviver sem controle → **comportamento imprevisível**

O Kubernetes oferece duas strategies nativas (`RollingUpdate` e `Recreate`) e suporta dois padrões arquiteturais implementados manualmente (`Blue-Green` e `Canary`).

---

## Como funciona internamente

### RollingUpdate (padrão)

A strategy padrão. O Deployment Controller orquestra a transição gradual entre o ReplicaSet antigo e o novo, respeitando dois parâmetros:

- **`maxUnavailable`**: quantos Pods podem ficar indisponíveis durante o rollout (pode ser número absoluto ou percentual)
- **`maxSurge`**: quantos Pods extras além do `replicas` desejado podem existir temporaneamente

```
Estado inicial:  [v1] [v1] [v1] [v1]  ← RS antigo, 4 réplicas

maxUnavailable=1, maxSurge=1:

Passo 1: cria 1 Pod v2 (surge)   → [v1][v1][v1][v1][v2]   (5 Pods, 0 indisponíveis)
Passo 2: termina 1 Pod v1         → [v1][v1][v1][v2]        (4 Pods, 1 indisponível momentâneo)
Passo 3: cria 1 Pod v2            → [v1][v1][v1][v2][v2]
Passo 4: termina 1 Pod v1         → [v1][v1][v2][v2]
... repete até:
Estado final:    [v2] [v2] [v2] [v2]  ← RS novo, RS antigo com 0 réplicas (mantido para rollback)
```

> [!info] O ReplicaSet antigo não é deletado
> Após o rollout, o RS antigo permanece com `replicas: 0`. Isso é o que permite `kubectl rollout undo`. O histórico é controlado por `spec.revisionHistoryLimit` (padrão: 10).

**Configuração zero-downtime conservadora** — nunca há Pod indisponível:
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0   # nunca mata antes de ter substituto pronto
      maxSurge: 1         # cria 1 extra, espera ficar Ready, aí mata 1 antigo
```

**Configuração rápida** — aceita indisponibilidade temporária:
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%  # até 25% dos Pods podem ser indisponíveis
      maxSurge: 25%        # até 25% de Pods extras
```

### Recreate

Simples e brutal. Todos os Pods do RS antigo são terminados *antes* de qualquer Pod novo ser criado.

```
Estado inicial:  [v1] [v1] [v1] [v1]

Passo 1: escala RS antigo para 0  → [] [] [] []  ← DOWNTIME COMEÇA AQUI
Passo 2: cria RS novo              → [v2][v2][v2][v2]  ← DOWNTIME TERMINA
```

```yaml
spec:
  strategy:
    type: Recreate
```

> [!warning] Recreate sempre causa downtime
> Use apenas quando duas versões não podem coexistir — ex: aplicações que fazem migrate de schema de banco de dados não compatível com a versão anterior.

### Blue-Green (padrão arquitetural)

Não é uma `spec.strategy` nativa — é implementado com dois Deployments e um Service que troca de seletor.

```
       ┌─────────────────────┐
       │    Service prod      │
       │  selector: slot=blue │ ← aponta para Blue
       └──────────┬──────────┘
                  │
    ┌─────────────┴──────────────┐
    │                            │
┌───▼───────┐           ┌───────────────┐
│  Blue (v1) │           │  Green (v2)   │
│  slot=blue │           │  slot=green   │
│  3 réplicas│           │  3 réplicas   │
│  (produção)│           │  (staging)    │
└───────────┘           └───────────────┘
```

A troca é **atômica**: um único `kubectl patch` muda o seletor do Service de `slot=blue` para `slot=green`. Em milissegundos, todo o tráfego vai para a nova versão.

**Vantagem:** rollback instantâneo — basta mudar o seletor de volta.  
**Desvantagem:** custo duplo de infraestrutura durante a janela de deploy.

### Canary (padrão arquitetural)

Expõe a nova versão para uma fração do tráfego enquanto a maioria ainda vai para a versão estável.

**Canary simples (por proporção de réplicas):**

```
Service selector: app=webapp  (sem distinção de versão)

Deployment stable:  8 réplicas  → recebe ~80% do tráfego (round-robin)
Deployment canary:  2 réplicas  → recebe ~20% do tráfego
```

**Canary com Ingress (controle preciso de tráfego):**

O nginx Ingress Controller suporta canary via annotations, independente do número de réplicas:

```yaml
# Ingress canary — annotation faz o nginx rotear X% para o serviço canary
nginx.ingress.kubernetes.io/canary: "true"
nginx.ingress.kubernetes.io/canary-weight: "20"   # 20% do tráfego
```

> [!tip] Canary por réplicas vs Canary por Ingress
> - **Por réplicas**: simples, sem dependência de Ingress, mas impreciso (round-robin pode variar). Bom para testes iniciais.
> - **Por Ingress**: controle exato de percentual, sem precisar escalar pods. Necessário quando você quer 1% ou 5% antes de abrir mais.

---

## Na prática — YAMLs e comandos

### RollingUpdate zero-downtime

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: default
spec:
  replicas: 4
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: webapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  minReadySeconds: 10          # aguarda 10s após Pod Ready antes de avançar
  progressDeadlineSeconds: 300 # rollout falha se não progredir em 5min
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: app
        image: nginx:1.27
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
        readinessProbe:        # CRÍTICO: sem readiness, o rollout não espera o Pod estar pronto
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

> [!warning] readinessProbe é obrigatória para RollingUpdate seguro
> Sem readinessProbe, o Kubernetes considera o Pod "Ready" assim que o container inicia — antes da aplicação estar de fato servindo tráfego. O tráfego vai para um Pod que ainda não está pronto. Sempre configure readinessProbe em Deployments com RollingUpdate.

### Recreate

```yaml
spec:
  strategy:
    type: Recreate
  # sem campos rollingUpdate
```

### Rollout controlado com pause/resume

```bash
# Iniciar um rollout e pausá-lo imediatamente
kubectl set image deployment/webapp app=nginx:1.27-alpine
kubectl rollout pause deployment/webapp

# Verificar estado — deve mostrar "paused"
kubectl rollout status deployment/webapp

# Inspecionar métricas, logs, health da versão canary que está rodando
kubectl get pods -l app=webapp

# Se estiver ok → continuar
kubectl rollout resume deployment/webapp

# Se não estiver → reverter
kubectl rollout undo deployment/webapp
```

### Blue-Green completo

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-blue
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      slot: blue
  template:
    metadata:
      labels:
        app: webapp
        slot: blue
    spec:
      containers:
      - name: app
        image: nginx:1.26
---
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-green
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      slot: green
  template:
    metadata:
      labels:
        app: webapp
        slot: green
    spec:
      containers:
      - name: app
        image: nginx:1.27
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
  namespace: default
spec:
  selector:
    app: webapp
    slot: blue     # ← começa apontando para blue
  ports:
  - port: 80
    targetPort: 80
```

```bash
# Verificar que o green está saudável antes de trocar
kubectl rollout status deployment/webapp-green
kubectl get pods -l slot=green

# Troca atômica: blue → green
kubectl patch service webapp -p '{"spec":{"selector":{"app":"webapp","slot":"green"}}}'

# Verificar
kubectl describe service webapp | grep Selector

# Rollback instantâneo se necessário
kubectl patch service webapp -p '{"spec":{"selector":{"app":"webapp","slot":"blue"}}}'
```

### Canary com Ingress weight

```yaml
# ingress-stable.yaml — Ingress principal (sem annotation canary)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-stable
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: webapp.staypuff.info
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-stable
            port:
              number: 80
---
# ingress-canary.yaml — Ingress canary com peso de 20%
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-canary
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
spec:
  ingressClassName: nginx
  rules:
  - host: webapp.staypuff.info
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-canary
            port:
              number: 80
```

```bash
# Aumentar gradualmente o peso do canary
kubectl annotate ingress webapp-canary nginx.ingress.kubernetes.io/canary-weight=50 --overwrite
kubectl annotate ingress webapp-canary nginx.ingress.kubernetes.io/canary-weight=100 --overwrite

# Promoção completa: deletar ingress canary e atualizar o stable
kubectl delete ingress webapp-canary
kubectl set image deployment/webapp-stable app=nginx:1.27
```

---

## Casos de uso e boas práticas

### Quando usar cada strategy

| Strategy | Use quando | Evite quando |
|---|---|---|
| **RollingUpdate** | Aplicação stateless, API REST, microsserviços | Schema de DB incompatível entre versões |
| **Recreate** | Migrations de banco incompatíveis, single-instance stateful | Há SLA de disponibilidade |
| **Blue-Green** | Deploy de alto risco com rollback instantâneo necessário, releases importantes | Custo de infra é restrição, state compartilhado entre versões |
| **Canary** | Feature flags, validar nova versão em produção com tráfego real, A/B testing | Aplicações stateful com sessão sticky |

### `minReadySeconds` — o parâmetro mais esquecido

```yaml
spec:
  minReadySeconds: 30
```

Sem `minReadySeconds`, o Kubernetes avança o rollout assim que o Pod fica `Ready` (readinessProbe passou). Mas "Ready" pode não significar "estável" — a aplicação pode ter um bug que se manifesta 10 segundos depois de iniciar.

`minReadySeconds: 30` faz o Deployment aguardar 30 segundos após o Pod estar Ready antes de continuar o rollout. Detecta crashes precoces.

### `progressDeadlineSeconds` — timeout do rollout

```yaml
spec:
  progressDeadlineSeconds: 300  # 5 minutos
```

Se o rollout não progredir em 5 minutos (ex: novo Pod nunca fica Ready), o Deployment é marcado como `False` em `Progressing` e o `kubectl rollout status` retorna exit code 1. Essencial para integrar em pipelines CI/CD.

### Checklist de deploy em produção

- [ ] readinessProbe configurada e testada
- [ ] `minReadySeconds` definido (≥ 10s para apps críticas)
- [ ] `progressDeadlineSeconds` definido e integrado ao CI
- [ ] `revisionHistoryLimit` configurado (5-10 é suficiente)
- [ ] PodDisruptionBudget criado para apps críticas
- [ ] Recursos (requests/limits) definidos corretamente

---

## Troubleshooting — cenários reais de produção

### Cenário 1: Rollout travado em `Progressing`

```bash
kubectl rollout status deployment/webapp
# Waiting for deployment "webapp" rollout to finish: 1 out of 4 new replicas have been updated...
# (trava aqui, não avança)

# 1. Ver eventos do Deployment
kubectl describe deployment webapp | tail -20

# 2. Ver os Pods que estão sendo criados
kubectl get pods -l app=webapp -w

# 3. Investigar Pod que não fica Ready
kubectl describe pod <pod-novo> | grep -A10 Events
# Causas comuns:
# - readinessProbe falhando → aplicação não inicia corretamente
# - ImagePullBackOff → imagem não existe ou registry inacessível
# - Pending → node sem recursos suficientes (requests muito altos)
# - OOMKilled → limite de memória muito baixo para a nova versão

# 4. Ver logs do container problemático
kubectl logs <pod-novo> --previous  # se crashou
kubectl logs <pod-novo>             # se ainda está tentando

# 5. Forçar rollback se necessário
kubectl rollout undo deployment/webapp
kubectl rollout status deployment/webapp  # aguardar rollback completar
```

### Cenário 2: Canary recebendo proporção errada de tráfego

```bash
# Canary com 2 réplicas e stable com 8 → esperado: 20% canary
# Mas está recebendo 50%?

# Verificar endpoints do Service
kubectl describe service webapp | grep Endpoints

# Se o Service não tem selector específico de versão, a distribuição é
# round-robin entre TODOS os Pods, proporção depende de réplicas.
# Verificar se os dois Deployments estão com réplicas corretas:
kubectl get deployment webapp-stable webapp-canary

# Lembrar: com kube-proxy/iptables, não é round-robin perfeito.
# Para controle preciso, use Ingress canary-weight.
```

### Cenário 3: Blue-Green — Service não trocou

```bash
# Patch foi aplicado mas o tráfego ainda vai para blue?

# 1. Verificar o seletor atual do Service
kubectl get service webapp -o jsonpath='{.spec.selector}'

# 2. Verificar se os Pods green têm os labels corretos
kubectl get pods -l slot=green --show-labels

# 3. Verificar endpoints — se vazio, o seletor não casa com nenhum Pod
kubectl describe service webapp | grep Endpoints
# Endpoints: <none>  ← seletor não casou

# 4. Comparar seletor do Service com labels dos Pods
# O seletor é case-sensitive e deve ser subconjunto dos labels do Pod
kubectl get pods -l app=webapp,slot=green
```

### Cenário 4: `maxUnavailable: 0` mas ainda tem downtime

```bash
# RollingUpdate com maxUnavailable=0 deveria ser zero-downtime,
# mas você está vendo 502s durante o deploy.

# Causa mais comum: Pod antigo recebe SIGTERM e é terminado antes de
# ser removido dos endpoints do Service.

# Solução: preStop hook para aguardar drenagem de conexões
spec:
  template:
    spec:
      containers:
      - name: app
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]  # aguarda kube-proxy atualizar iptables
      terminationGracePeriodSeconds: 30
```

---

## Nível avançado — Argo Rollouts e automação

### PodDisruptionBudget — proteção durante deploys

Um PDB garante que durante um rollout (ou node drain), nunca haja menos do que N Pods disponíveis:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: webapp-pdb
  namespace: default
spec:
  minAvailable: 2     # ou maxUnavailable: 1
  selector:
    matchLabels:
      app: webapp
```

```bash
# Ver PDBs e seu estado atual
kubectl get pdb
kubectl describe pdb webapp-pdb
```

### Argo Rollouts — strategies avançadas com análise automática

O Argo Rollouts é um CRD que substitui o `Deployment` para casos onde você precisa de:
- Canary com promoção automática baseada em métricas (Prometheus, Datadog)
- Blue-Green com análise automática antes de promover
- `pause` e `promote` via CLI

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: webapp
spec:
  replicas: 4
  strategy:
    canary:
      steps:
      - setWeight: 10      # 10% do tráfego
      - pause: {duration: 1m}
      - setWeight: 25
      - pause: {duration: 2m}
      - analysis:          # verifica métricas antes de continuar
          templates:
          - templateName: error-rate
      - setWeight: 100
```

```bash
# Comandos do kubectl argo rollouts plugin
kubectl argo rollouts get rollout webapp --watch
kubectl argo rollouts promote webapp     # avança para próximo step
kubectl argo rollouts pause webapp       # pausa
kubectl argo rollouts abort webapp       # aborta e reverte
```

### Comparação: estratégias de implementação

| Abordagem | Controle de tráfego | Complexidade | Rollback | Análise automática |
|---|---|---|---|---|
| RollingUpdate nativo | Por réplicas | Baixa | `rollout undo` | Não |
| RollingUpdate + pause | Manual por réplicas | Baixa | `rollout undo` | Não |
| Blue-Green manual | Atômico (Service swap) | Média | `patch` no Service | Não |
| Ingress Canary | Por percentual exato | Média | `delete` ingress canary | Não |
| Argo Rollouts | Por percentual + métricas | Alta | `abort` | Sim |
| Flagger | Automático por métricas | Alta | Automático | Sim |

---

## Referências

- **Kubernetes docs** — [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- **Kubernetes docs** — [Rolling Update](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)
- **nginx Ingress** — [Canary Annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary)
- **Argo Rollouts** — [argoproj.github.io/argo-rollouts](https://argoproj.github.io/argo-rollouts/)
- **Flagger** — [flagger.app](https://flagger.app/)
- **Linuxtips DK8S** — `/mnt/c/Users/felip/Documents/Kubernets/DK8S`
- **Martin Fowler** — [BlueGreenDeployment](https://martinfowler.com/bliki/BlueGreenDeployment.html)
