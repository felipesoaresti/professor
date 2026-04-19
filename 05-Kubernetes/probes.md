---
tags:
  - kubernetes
  - probes
  - readiness
  - liveness
  - startup
  - reliability
area: kubernetes
tipo: conteudo
prerequisites:
  - "[[05-Kubernetes/kubernetes-teoria-inicial]]"
  - "[[05-Kubernetes/deployment-strategies]]"
next:
  - "[[11-Exercicios/probes]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Probes em Deployments

> Conteúdo: [[05-Kubernetes/probes]] | Exercícios: [[11-Exercicios/probes]] | Trilha: [[00-Trilha/kubernetes]]

Probes são o mecanismo pelo qual o Kubernetes sabe se um container está **pronto para receber tráfego**, **ainda saudável**, ou **ainda inicializando**. Sem probes, o Kubernetes opera cego — assume que tudo está bem apenas porque o processo está rodando. Isso causa dois tipos de falha em produção: tráfego indo para Pods que ainda não estão prontos, e Pods travados que nunca são reiniciados.

---

## O que é e por que existe

O Kubernetes tem três probes, cada uma com um papel distinto:

| Probe | Pergunta | Consequência de falha |
|---|---|---|
| **readinessProbe** | "O Pod está pronto para receber tráfego?" | Remove o Pod dos Endpoints do Service (sem restart) |
| **livenessProbe** | "O Pod ainda está funcionando?" | Reinicia o container |
| **startupProbe** | "O container terminou de inicializar?" | Bloqueia liveness e readiness até passar |

**Sem readinessProbe:** durante um RollingUpdate, o Kubernetes manda tráfego para o Pod novo assim que o container inicia — antes da aplicação estar servindo. Resultado: 502/503 durante deploys.

**Sem livenessProbe:** se a aplicação trava em deadlock ou loop infinito (processo ainda vivo, porta ainda aberta, mas sem processar requisições), o Kubernetes não sabe. O Pod fica no ar mas inoperante indefinidamente.

**Sem startupProbe:** aplicações com inicialização lenta (JVM, índices de banco) precisam de `initialDelaySeconds` alto na livenessProbe. Mas `initialDelaySeconds` alto também atrasa a detecção de falhas pós-inicialização. A startupProbe resolve isso separando os dois problemas.

---

## Como funciona internamente

### O kubelet é o executor das probes

Probes não são executadas pelo container ou pelo API server — são executadas pelo **kubelet** do node onde o Pod roda. O kubelet executa cada probe no intervalo definido e interpreta o resultado:

- **Sucesso (0 / HTTP 2xx-3xx / porta aberta):** probe passou
- **Falha (non-zero / HTTP 4xx-5xx / porta fechada / timeout):** probe falhou
- **Erro:** probe falhou (ex: exec retornou erro, conexão recusada)

### O ciclo de vida das probes

```
Container inicia
      │
      ▼
[startupProbe] ← se definida, bloqueia as demais
      │ passa (ou não definida)
      ▼
[readinessProbe] ─── falha → remove dos Endpoints (sem restart)
      │                           │ passa → re-adiciona
      ▼
[livenessProbe]  ─── falha × failureThreshold → SIGTERM → reinicia container
```

A readiness e liveness rodam **em paralelo e continuamente** após a startup passar. Um Pod pode passar de Ready para NotReady e voltar várias vezes sem reiniciar — isso é readiness. Um Pod só reinicia quando a liveness falha `failureThreshold` vezes consecutivas.

### Parâmetros universais

```yaml
probe:
  initialDelaySeconds: 5    # aguarda N segundos após o container iniciar antes da primeira probe
  periodSeconds: 10         # intervalo entre probes (padrão: 10s)
  timeoutSeconds: 1         # timeout de cada probe (padrão: 1s)
  successThreshold: 1       # quantas successes consecutivas para considerar OK (padrão: 1)
  failureThreshold: 3       # quantas falhas consecutivas antes de agir (padrão: 3)
```

> [!warning] successThreshold na livenessProbe deve ser 1
> A liveness só aceita `successThreshold: 1`. Valores maiores são ignorados. Para readiness, `successThreshold > 1` faz sentido em apps que precisam "estabilizar" após um breve período de warmup.

### Os 4 mecanismos de probe

#### httpGet
O kubelet faz uma requisição HTTP ao container. Qualquer status 2xx ou 3xx é sucesso.

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: X-Custom-Header
      value: probe
  periodSeconds: 5
  timeoutSeconds: 3
```

#### tcpSocket
O kubelet tenta abrir uma conexão TCP na porta. Conexão estabelecida = sucesso.

```yaml
livenessProbe:
  tcpSocket:
    port: 5432       # útil para PostgreSQL, Redis — sem endpoint HTTP
  periodSeconds: 10
```

#### exec
O kubelet executa um comando dentro do container. Exit code 0 = sucesso.

```yaml
livenessProbe:
  exec:
    command:
    - sh
    - -c
    - "pg_isready -U postgres"
  periodSeconds: 10
  timeoutSeconds: 5
```

#### grpc (Kubernetes 1.24+)
O kubelet usa o protocolo gRPC Health Checking. Útil para serviços gRPC sem endpoint HTTP.

```yaml
readinessProbe:
  grpc:
    port: 9090
    service: ""      # string vazia = health check geral
  periodSeconds: 10
```

---

## Na prática — YAMLs e exemplos reais

### Configuração completa para aplicação web (o padrão de referência)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
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

        startupProbe:           # aguarda inicialização sem prejudicar liveness
          httpGet:
            path: /healthz
            port: 80
          failureThreshold: 30  # 30 × 10s = até 5 minutos para inicializar
          periodSeconds: 10

        readinessProbe:         # controla se o Pod recebe tráfego
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 0  # startup já fez o trabalho de esperar
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3   # 3 × 5s = 15s sem tráfego antes de remover dos endpoints

        livenessProbe:          # detecta travamentos
          httpGet:
            path: /healthz
            port: 80
          periodSeconds: 15
          timeoutSeconds: 5
          failureThreshold: 3   # 3 × 15s = 45s em falha antes de reiniciar
```

### Separando `/healthz` de `/ready` — o padrão correto

- **`/healthz` (liveness):** responde 200 se o processo está vivo e não travado. Deve ser ultra-simples — sem checar dependências externas. Só verifica o estado interno da aplicação.
- **`/ready` (readiness):** responde 200 se a aplicação está pronta para servir tráfego. Pode checar conexão com banco de dados, cache, aquecimento de dados.

> [!warning] Nunca cheque dependências externas na livenessProbe
> Se a liveness checa conexão com banco e o banco fica lento, a liveness falha → container reinicia → novo container tenta conectar no banco lento → falha → reinicia → loop infinito (CrashLoopBackOff). A dependência externa fica sobrecarregada com reconexões. Use dependências externas apenas na readiness.

### Aplicação Java/JVM com inicialização lenta

```yaml
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  failureThreshold: 60    # 60 × 10s = até 10 minutos para JVM aquecer
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  periodSeconds: 10
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  periodSeconds: 20
  failureThreshold: 3
```

> [!info] Spring Boot Actuator
> Spring Boot 2.3+ expõe `/actuator/health/readiness` e `/actuator/health/liveness` nativamente, seguindo o padrão Kubernetes. Habilitar com `management.health.probes.enabled=true`.

### PostgreSQL — sem endpoint HTTP

```yaml
livenessProbe:
  exec:
    command:
    - sh
    - -c
    - pg_isready -U $(POSTGRES_USER) -d $(POSTGRES_DB)
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 6

readinessProbe:
  exec:
    command:
    - sh
    - -c
    - pg_isready -U $(POSTGRES_USER) -d $(POSTGRES_DB)
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
```

### Redis — tcpSocket

```yaml
livenessProbe:
  tcpSocket:
    port: 6379
  periodSeconds: 10
  timeoutSeconds: 1
  failureThreshold: 3

readinessProbe:
  exec:
    command:
    - sh
    - -c
    - redis-cli ping | grep PONG
  periodSeconds: 5
  failureThreshold: 3
```

### Observando probes em ação

```bash
# Ver estado das probes de um Pod
kubectl describe pod <nome> | grep -A15 "Containers:"
# Campos: Liveness, Readiness, Startup — com configuração e último resultado

# Ver eventos relacionados a probes
kubectl describe pod <nome> | grep -A5 Events
# "Liveness probe failed" → container vai ser reiniciado
# "Readiness probe failed" → removido dos endpoints
# "Startup probe failed" → aguardando inicialização

# Monitorar em tempo real se um Pod está Ready
kubectl get pod <nome> -w
# Coluna READY: 0/1 → sem tráfego; 1/1 → recebendo tráfego

# Verificar se o Pod está nos Endpoints do Service
kubectl describe service <svc> | grep Endpoints
kubectl get endpoints <svc>
```

---

## Casos de uso e boas práticas

### Regra de ouro: 3 endpoints, 3 probes

| Endpoint | Probe | O que checa |
|---|---|---|
| `GET /healthz` | livenessProbe | Processo vivo, não em deadlock, sem estado corrompido |
| `GET /ready` | readinessProbe | Banco conectado, cache aquecido, pronto para servir |
| `GET /startup` ou reusar `/healthz` | startupProbe | Container passou da fase de inicialização |

### Valores recomendados para web APIs

```yaml
startupProbe:
  failureThreshold: 30   # app tem até 5min para iniciar
  periodSeconds: 10

readinessProbe:
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3    # remove em 15s de falha

livenessProbe:
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 3    # reinicia em 45s de falha
```

### PodDisruptionBudget + readinessProbe

O PDB garante que durante um `kubectl drain` ou RollingUpdate, nunca haja menos que N Pods Ready. "Ready" aqui depende da readinessProbe — Pods com readiness falhando não contam para o mínimo do PDB.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: webapp-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: webapp
```

### terminationGracePeriodSeconds e probes

Quando a liveness falha e o container vai ser reiniciado, o kubelet envia SIGTERM e aguarda `terminationGracePeriodSeconds` (padrão: 30s) antes de enviar SIGKILL. Durante esse tempo, a readinessProbe continua rodando — e normalmente falha, removendo o Pod dos Endpoints antes do SIGKILL chegar.

```yaml
spec:
  terminationGracePeriodSeconds: 60  # dá 60s para o container finalizar conexões
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["sleep", "5"]  # aguarda kube-proxy propagar remoção dos endpoints
```

---

## Troubleshooting — cenários reais de produção

### Cenário 1: CrashLoopBackOff causado por livenessProbe mal configurada

```bash
# Sintoma: Pod reinicia repetidamente logo após iniciar
kubectl get pods
# NAME       READY   STATUS             RESTARTS   AGE
# webapp-x   0/1     CrashLoopBackOff   8          15m

kubectl describe pod webapp-x | grep -A10 Events
# Warning  Unhealthy  Liveness probe failed: Get "http://10.x.x.x:8080/healthz":
#          dial tcp 10.x.x.x:8080: connect: connection refused

# Causa: livenessProbe roda antes da app estar pronta para responder
# (sem startupProbe, initialDelaySeconds muito baixo)

# Diagnóstico: ver quando a app fica pronta nos logs
kubectl logs webapp-x --previous
# "Application started in 45.3s" → mas initialDelaySeconds era 10

# Solução: adicionar startupProbe com failureThreshold adequado
# OU aumentar initialDelaySeconds da liveness
```

### Cenário 2: Tráfego indo para Pod que retorna 503

```bash
# Sintoma: alguns requests retornam 503 durante o deploy
# Causa: sem readinessProbe, o Pod entra nos Endpoints antes de estar pronto

# Verificar se há readinessProbe configurada
kubectl describe pod <pod-novo> | grep Readiness
# (vazio = sem readinessProbe)

# Verificar quando o Pod foi adicionado aos Endpoints
kubectl get endpoints <svc> -w
# O IP do Pod aparece imediatamente quando fica Running,
# mas a aplicação ainda está inicializando

# Solução: adicionar readinessProbe e remover o Pod dos Endpoints
# até a aplicação estar de fato pronta
```

### Cenário 3: Pod oscilando entre Ready e NotReady

```bash
# Sintoma: kubectl get pods mostra READY 0/1 intermitentemente
kubectl get pods -w
# webapp-x   1/1     Running   0   5m
# webapp-x   0/1     Running   0   5m   ← readiness falhou
# webapp-x   1/1     Running   0   6m   ← readiness voltou

# Verificar frequência e causa
kubectl describe pod webapp-x | grep -E "Readiness|Warning"

# Possíveis causas:
# 1. timeoutSeconds muito baixo — a app responde em 2s mas timeout é 1s
kubectl describe pod webapp-x | grep "timeout"
# Solução: aumentar timeoutSeconds

# 2. App legítimamente lenta em momentos de carga
# Verificar correlação com CPU/memory usage
kubectl top pod webapp-x

# 3. readinessProbe checando dependência instável (ex: banco com timeout ocasional)
# Solução: separar endpoint de readiness de dependências externas instáveis
```

### Cenário 4: RollingUpdate não avança (trava em 1/4 atualizado)

```bash
# Deployment tem maxUnavailable:0, maxSurge:1
# O Pod novo é criado mas nunca fica Ready → rollout para

kubectl rollout status deployment/webapp
# Waiting for deployment "webapp" rollout to finish:
# 1 out of 4 new replicas have been updated...

# O Pod novo não está Ready porque a readinessProbe está falhando
kubectl get pods -l app=webapp
# webapp-new-xxx   0/1   Running   0   5m  ← nunca vai para 1/1

kubectl describe pod webapp-new-xxx | tail -20
# Warning  Unhealthy  Readiness probe failed

# Opções:
# 1. Corrigir o código/config que faz a readiness falhar → novo deploy
# 2. Rollback
kubectl rollout undo deployment/webapp
```

### Cenário 5: startupProbe expirando em app lenta

```bash
# Sintoma: Pod reinicia durante inicialização, antes de estar pronto
kubectl describe pod <pod> | grep -A5 Events
# Warning  Unhealthy  Startup probe failed after X attempts

# O failureThreshold × periodSeconds não é suficiente para o tempo de startup

# Calcular: se app demora 8min para iniciar
# failureThreshold: 48  × periodSeconds: 10 = 480s = 8min de tolerância

# Ajustar o Deployment com novo failureThreshold
kubectl patch deployment webapp --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/startupProbe/failureThreshold","value":48}]'
```

---

## Nível avançado — edge cases e CKA/Staff

### A interação exata entre as três probes

```
t=0:   container inicia
t=0:   startupProbe começa a rodar (periodSeconds: 10)
t=10:  startupProbe falha (app ainda iniciando)  → contador: 1/30
t=20:  startupProbe falha                        → contador: 2/30
...
t=120: startupProbe passa! → liveness e readiness começam
t=120: readinessProbe roda → falha (failureThreshold: 3) → contador: 1/3
t=125: readinessProbe roda → passa  → Pod entra nos Endpoints
t=135: livenessProbe roda → passa
```

Enquanto a startupProbe não passa, a livenessProbe **não reinicia** o container mesmo que estivesse configurada para falhar. Isso é a proteção que a startupProbe oferece.

### Probe overhead e impacto em performance

Cada probe é uma requisição extra ao container. Com 3 probes e `periodSeconds: 5`:
- 12 requisições por minuto por container
- Em um Deployment com 10 réplicas: 120 requisições/minuto apenas de probes
- Se `timeoutSeconds: 10` e a app é lenta: goroutines/threads acumulando

Para apps com alta carga: aumentar `periodSeconds` das probes para reduzir overhead.

### Probe failures no contexto de QoS e OOM

Se o container é OOM-killed durante uma probe httpGet, o kubelet recebe connection refused e conta como falha de probe. Se isso acontecer `failureThreshold` vezes → reinicia o container. Mas a causa raiz é o OOM, não a probe. Sempre verificar `memory.events` do cgroup antes de culpar a probe.

```bash
# Verificar se há OOM antes de investigar probe
kubectl describe pod <pod> | grep -E "OOMKilled|Reason"
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState}'
```

### Exec probe e processos zumbi

Quando a `exec` probe usa `/bin/sh -c`, cada execução cria um processo filho. Se o shell não fizer cleanup adequado ou se o timeoutSeconds for excedido antes do exec retornar, podem acumular processos zumbi no container. Em containers com `pid: 1` customizado (sem init system), isso pode esgotar a tabela de PIDs.

```bash
# Verificar processos dentro do container
kubectl exec <pod> -- ps aux | grep defunct
```

### Configuração de probe via Downward API

É possível usar variáveis de ambiente para configurar thresholds dinamicamente, mas probes não aceitam variáveis — os valores são literais no spec. A alternativa é um wrapper script:

```yaml
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - |
      curl -sf http://localhost:8080/healthz || exit 1
  periodSeconds: 10
```

### gRPC probe e requisitos

A `grpc` probe requer que o container implemente o [gRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md). O kubelet usa o binário `grpc-health-probe` internamente a partir do Kubernetes 1.24. Se o serviço não implementa o protocolo, a probe sempre falha.

---

## Referências

- **Kubernetes docs** — [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- **Kubernetes docs** — [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- **gRPC Health Checking Protocol** — github.com/grpc/grpc/blob/master/doc/health-checking.md
- **Spring Boot Actuator** — docs.spring.io/spring-boot/docs/current/reference/html/actuator.html
- **Linuxtips DK8S** — `/mnt/c/Users/felip/Documents/Kubernets/DK8S`
- `man kubectl-describe` — eventos de probe no output de describe
