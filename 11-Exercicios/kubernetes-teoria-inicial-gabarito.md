---
tags:
  - exercicios
  - kubernetes
  - gabarito
  - teoria
  - arquitetura
tipo: gabarito
area: kubernetes
conteudo: "[[05-Kubernetes/kubernetes-teoria-inicial]]"
exercicios: "[[11-Exercicios/kubernetes-teoria-inicial]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Gabarito — Kubernetes Teoria Inicial

> Exercícios: [[11-Exercicios/kubernetes-teoria-inicial]] | Conteúdo: [[05-Kubernetes/kubernetes-teoria-inicial]] | Trilha: [[00-Trilha/kubernetes]]

> [!info] Como usar este gabarito
> Leia **depois** de ter tentado o exercício. Cada seção descreve o que a saída correta deve conter, as armadilhas mais comuns e o que o avaliador (ou o exame CKA) estaria verificando.

---

## Exercício 1 — Reconhecimento do cluster

### O que a saída correta mostra

**`kubectl get nodes -o wide`**
```
NAME         STATUS   ROLES           AGE   VERSION   INTERNAL-IP     OS-IMAGE
k8s-master   Ready    control-plane   Xd    v1.35.3   192.168.3.30    Debian GNU/Linux 13
k8s-worker1  Ready    <none>          Xd    v1.35.3   192.168.3.31    Debian GNU/Linux 13
k8s-worker2  Ready    <none>          Xd    v1.35.3   192.168.3.32    Debian GNU/Linux 13
```

Pontos críticos:
- Os 3 nodes estão `Ready`. Se algum estiver `NotReady`, há um problema sério — **pare tudo e investigue antes de continuar**.
- `ROLES` do master mostra `control-plane`. Workers mostram `<none>` (padrão — o role `worker` não é atribuído automaticamente).
- A coluna `VERSION` deve ser idêntica nos 3 nodes. Versões diferentes indicam upgrade parcial.

**`kubectl describe node k8s-worker1`**

Seção crítica que você deveria ter lido:
```
Conditions:
  MemoryPressure     False
  DiskPressure       False
  PIDPressure        False
  Ready              True

Taints:             <none>

Allocated resources:
  Resource           Requests    Limits
  --------           --------    ------
  cpu                XXXm (X%)   XXXm (X%)
  memory             XXXMi (X%)  XXXMi (X%)
```

> [!warning] Armadilha comum
> Ignorar a seção `Taints`. No homelab, o control-plane tem taint `node-role.kubernetes.io/control-plane:NoSchedule`. Isso significa que Pods comuns **nunca serão agendados no master** sem uma toleration explícita — e isso é o comportamento correto.

**`kubectl top nodes`** — requer metrics-server instalado. Se der erro `metrics not available`, o metrics-server não está rodando no cluster. Não é um problema do exercício, é uma dependência de infraestrutura.

### O que o avaliador verificaria
- Você consegue identificar o IP de cada node sem hesitar?
- Você leu `Allocated resources` e entendeu a diferença entre o que está **solicitado** (requests) vs o que está **sendo usado** (top)?
- Você notou que o master tem taint?

---

## Exercício 2 — Primeiro Pod e namespace

### Comandos corretos

```bash
# Criar namespace
sudo kubectl create namespace estudo

# Criar Pod (modo imperativo — válido para CKA)
sudo kubectl run meu-nginx \
  --image=nginx:1.27 \
  --labels="app=meu-nginx" \
  -n estudo

# Ou via manifest:
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: meu-nginx
  namespace: estudo
  labels:
    app: meu-nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.27
```

### Respostas esperadas

**Em qual node foi para?**
O scheduler distribui baseado em recursos disponíveis. No homelab com 2 workers, provavelmente foi para worker1 ou worker2 — nunca para o master (taint). Não há garantia de qual worker sem nodeSelector.

**`curl localhost` dentro do container:**
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```
A página padrão do nginx. Isso prova que o processo está rodando e escutando na porta 80 dentro do container.

**O IP muda após deletar e recriar?**

**Sim, sempre.** O IP de Pod é efêmero — alocado pelo CNI (Calico no homelab) no momento da criação. Mesmo com o mesmo nome, o Pod recriado recebe um IP diferente do pool. Isso é o problema que Services existem para resolver.

> [!tip] Insight fundamental
> `kubectl delete pod meu-nginx -n estudo` e depois criar com o mesmo nome demonstra uma verdade central: **o nome é estável, o IP não**. Services abstraem o IP instável expondo um IP virtual (ClusterIP) que permanece fixo.

### Armadilhas
- `kubectl run` sem `--restart=Never` cria um Pod com `restartPolicy: Always` — correto para este exercício, mas para Jobs use `--restart=Never`.
- `kubectl exec -it meu-nginx -n estudo -- /bin/bash` pode falhar se o nginx usar imagem slim sem bash. Use `/bin/sh` ou adicione `bash` via `apt`.

---

## Exercício 3 — Deployment com réplicas e self-healing

### Manifest correto

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: estudo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
```

### Respostas esperadas

**Distribuição entre workers:**
Com 3 réplicas e 2 workers, a distribuição típica é 2+1 ou 1+2, nunca 3+0 (o scheduler balanceia por padrão). Se todos forem para o mesmo worker, há algo errado (taint, resources insuficientes no outro).

**Tempo para recriar após delete manual:**
Tipicamente **2–5 segundos**. O controller loop do ReplicaSet detecta a divergência entre estado desejado (3) e atual (2) e age imediatamente. O tempo é dominado por: image pull (se não em cache) + CNI allocation + container start.

**Ao deletar 2 simultaneamente:**
O ReplicaSet recria os 2 em paralelo — não sequencial. Você verá 2 Pods em `ContainerCreating` ao mesmo tempo.

**Scale para 5 → depois para 2 — quais são terminados?**

Os **mais novos** são terminados primeiro (Last-In-First-Out por padrão). O Kubernetes usa o índice de criação para determinar a ordem de remoção em Deployments. Você pode verificar com `kubectl get pods -n estudo --sort-by=.metadata.creationTimestamp`.

> [!warning] Armadilha crítica de produção
> `requests` ≠ `limits`. Sem `requests` definido, o Pod entra na classe QoS `BestEffort` — o primeiro a ser evicted sob pressão de memória. Sem `limits`, um Pod pode consumir toda a memória do node e causar OOM no node. **Sempre defina requests. Sempre defina limits para memória.**

### O que verificar

```bash
# QoS class do Pod — deve ser Burstable (requests != limits)
sudo kubectl get pod <pod-name> -n estudo -o jsonpath='{.status.qosClass}'
# Output: Burstable
```

Se requests == limits, a classe seria `Guaranteed` — melhor para latência crítica, pior para densidade.

---

## Exercício 4 — Service e acesso interno

### Comandos corretos

```bash
# Criar Service ClusterIP (modo imperativo)
sudo kubectl expose deployment webapp \
  --name=webapp-svc \
  --port=80 \
  --target-port=80 \
  -n estudo

# Ou manifest:
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
  namespace: estudo
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
```

### Respostas esperadas

**Endpoints com 3 IPs:**
```bash
sudo kubectl get endpoints webapp-svc -n estudo
# NAME         ENDPOINTS                                      AGE
# webapp-svc   10.x.x.1:80,10.x.x.2:80,10.x.x.3:80         Xs
```
Cada IP corresponde a um Pod. O kube-proxy (ou eBPF no Calico) cria regras iptables/eBPF que balanceiam o tráfego entre esses IPs em round-robin.

**DNS dentro do cluster:**
O formato completo é `<service>.<namespace>.svc.cluster.local`. O CoreDNS resolve isso para o ClusterIP do Service. De dentro do mesmo namespace, `webapp-svc` sozinho também resolve (via search domain).

**Após deletar 1 Pod — Endpoint some e volta?**
Sim. O Endpoint controller remove o IP do Pod deletado em segundos e adiciona o IP do novo Pod assim que ele fica `Running` e passa nas readiness probes (se existir). Você veria:
1. Delete Pod → Endpoint cai para 2 IPs
2. Pod recriado → Endpoint volta para 3 IPs

> [!info] EndpointSlice vs Endpoints
> Desde K8s 1.21+, o objeto moderno é `EndpointSlice` (escala melhor em clusters grandes). `kubectl get endpoints` ainda funciona mas internamente usa EndpointSlice. No CKA, ambos são aceitos.

**Port-forward:**
```bash
sudo kubectl port-forward svc/webapp-svc 8080:80 -n estudo
# Forwarding from 127.0.0.1:8080 -> 80
```
Port-forward **não usa o load balancer do Service** — conecta diretamente a um Pod específico. É para debug apenas, não para produção.

---

## Exercício 5 — Rollout e rollback

### Fluxo correto

```bash
# Atualizar imagem
sudo kubectl set image deployment/webapp nginx=nginx:1.27-alpine -n estudo

# Acompanhar em tempo real
sudo kubectl rollout status deployment/webapp -n estudo

# Histórico
sudo kubectl rollout history deployment/webapp -n estudo

# Forçar falha
sudo kubectl set image deployment/webapp nginx=nginx:versao-que-nao-existe -n estudo

# Ver eventos
sudo kubectl get events -n estudo --sort-by='.lastTimestamp' | tail -15

# Rollback
sudo kubectl rollout undo deployment/webapp -n estudo

# Rollback para revisão específica
sudo kubectl rollout undo deployment/webapp --to-revision=1 -n estudo
```

### Respostas esperadas

**Os Pods antigos continuam rodando durante a imagem inválida?**

**Sim** — e isso é a estratégia `RollingUpdate` funcionando corretamente. Com `maxUnavailable: 25%` e `maxSurge: 25%` (padrão), o Deployment garante que pelo menos 75% dos Pods estejam disponíveis. Quando o novo Pod não consegue puxar a imagem (`ImagePullBackOff`), o rollout para e os Pods antigos permanecem rodando.

```bash
sudo kubectl get deployment webapp -n estudo
# NAME     READY   UP-TO-DATE   AVAILABLE
# webapp   3/3     1            3
# UP-TO-DATE=1 significa 1 Pod tentando atualizar, mas AVAILABLE=3 (os antigos ainda servem tráfego)
```

> [!warning] Armadilha
> `rollout undo` **sem `--to-revision`** reverte apenas 1 revisão para trás. Se você fez 5 rollouts, `undo` vai para a revisão 4, não para a 1. Para revisões específicas, sempre use `--to-revision=N`.

**`rollout history` após exercício:**
```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```
`CHANGE-CAUSE` fica `<none>` quando não se usa `--record` (deprecated) ou a annotation `kubernetes.io/change-cause`. Em produção, documente o motivo do rollout diretamente na annotation:
```bash
sudo kubectl annotate deployment webapp kubernetes.io/change-cause="update para nginx:1.27-alpine — ticket INFRA-123" -n estudo
```

> [!tip] Débito técnico identificado
> O `rollout undo` do exercício original pode não ter sido executado antes de prosseguir para o Ex4. Se o Deployment ainda está com `nginx:versao-que-nao-existe`, corrija agora:
> ```bash
> sudo kubectl rollout undo deployment/webapp -n estudo
> sudo kubectl rollout status deployment/webapp -n estudo
> ```

---

## Exercício 6 — Labels e selectors

### Comandos corretos

```bash
# Service com selector correto
sudo kubectl create service clusterip mismatch-svc \
  --tcp=80:80 -n estudo
# Depois editar o selector via patch:
sudo kubectl patch svc mismatch-svc -n estudo \
  -p '{"spec":{"selector":{"app":"correto"}}}'

# Pods com label errado
sudo kubectl run pod-errado-1 --image=nginx:1.27 \
  --labels="app=errado" -n estudo
sudo kubectl run pod-errado-2 --image=nginx:1.27 \
  --labels="app=errado" -n estudo

# Corrigir labels SEM deletar os Pods
sudo kubectl label pod pod-errado-1 app=correto --overwrite -n estudo
sudo kubectl label pod pod-errado-2 app=correto --overwrite -n estudo

# Adicionar label de versão em apenas um
sudo kubectl label pod pod-errado-1 version=1.0 -n estudo

# Service que filtra por versão
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: version-svc
  namespace: estudo
spec:
  selector:
    app: correto
    version: "1.0"
  ports:
  - port: 80
```

### Respostas esperadas

**Antes da correção:**
```bash
sudo kubectl get endpoints mismatch-svc -n estudo
# NAME          ENDPOINTS   AGE
# mismatch-svc  <none>      Xs
```

**Após `kubectl label --overwrite`:**
```bash
sudo kubectl get endpoints mismatch-svc -n estudo
# NAME          ENDPOINTS                       AGE
# mismatch-svc  10.x.x.1:80,10.x.x.2:80        Xs
```

**`version-svc` com apenas 1 Pod:**
```bash
sudo kubectl get endpoints version-svc -n estudo
# NAME          ENDPOINTS       AGE
# version-svc   10.x.x.1:80    Xs
```

> [!tip] Insight fundamental
> Labels são a "cola" do Kubernetes. O selector do Service não referencia o nome do Pod — referencia labels. Isso permite:
> - Mover Pods entre Services sem recriá-los
> - Canary deployments (adicionar label `track=canary` em subset de Pods)
> - Blue/green via simples troca de selector no Service

> [!warning] Armadilha comum
> `kubectl label pod pod-errado-1 app=correto` **sem `--overwrite`** falha com erro se o label já existe. Sempre use `--overwrite` ao corrigir labels existentes.

---

## Exercício 7 — kubectl explain

### Fluxo correto de pesquisa

```bash
# Estrutura top-level
sudo kubectl explain pod
# Campos: apiVersion, kind, metadata, spec, status

sudo kubectl explain pod.spec
# containers, initContainers, volumes, restartPolicy, ...

sudo kubectl explain pod.spec.containers.resources
# limits, requests — ambos são map[string]resource.Quantity

sudo kubectl explain pod.spec.containers.readinessProbe
# exec, httpGet, tcpSocket, initialDelaySeconds, periodSeconds, ...

sudo kubectl explain pod.spec.containers.readinessProbe.httpGet
# host, httpHeaders, path, port, scheme
```

### Pod YAML correto com todos os requisitos

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-explain
  namespace: estudo
spec:
  restartPolicy: Never
  containers:
  - name: nginx
    image: nginx:1.27
    resources:
      requests:
        cpu: 50m
        memory: 32Mi
      limits:
        cpu: 100m
        memory: 64Mi
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
```

### O que verificar

```bash
sudo kubectl get pod pod-explain -n estudo
# READY: 1/1, STATUS: Running
# READY 1/1 confirma que a readinessProbe passou
```

Se o Pod estiver `Running` mas `READY 0/1`, a readinessProbe está falhando. Causas comuns:
- Path incorreto (nginx responde em `/` mas a probe aponta para `/health`)
- Porta incorrena
- `initialDelaySeconds` muito curto (app ainda inicializando)

> [!tip] CKA — use `kubectl explain` como documentação offline
> No exame: `kubectl explain pod.spec.containers --recursive` mostra toda a árvore de campos. Com `| grep -A2 <campo>` você localiza qualquer subcampo sem navegar na documentação. Treine isso até virar reflexo.

---

## Exercício 8 — Troubleshooting sistemático

### Método padrão de diagnóstico (memorize esta sequência)

```
1. kubectl get pods -n estudo          → qual o STATUS?
2. kubectl describe pod <nome> -n estudo → qual o EVENTO/CONDIÇÃO?
3. kubectl logs <nome> -n estudo        → o que o processo reportou?
4. kubectl logs <nome> -n estudo --previous → se reiniciou, o que disse antes?
5. kubectl get events -n estudo --sort-by='.lastTimestamp' → timeline completa
```

### Manifest 1 — ImagePullBackOff

**Sintoma** (`kubectl get pods`):
```
NAME            READY   STATUS             RESTARTS   AGE
app-sem-imagem  0/1     ImagePullBackOff   0          Xm
```

**Causa raiz** (`kubectl describe pod app-sem-imagem -n estudo`):
```
Events:
  Warning  Failed     Xs    kubelet  Failed to pull image "nginx:versao-que-nao-existe-2099": ...
  Warning  Failed     Xs    kubelet  Error: ErrImagePull
  Normal   BackOff    Xs    kubelet  Back-off pulling image "nginx:versao-que-nao-existe-2099"
```

**Correção:**
```bash
sudo kubectl set image pod/app-sem-imagem app=nginx:1.27 -n estudo
# Nota: set image em Pod não funciona — você precisa deletar e recriar
sudo kubectl delete pod app-sem-imagem -n estudo
# Recriar com imagem válida
```

> [!info] Por que `set image` não funciona em Pod?
> `kubectl set image` funciona em Deployments (que controlam o template). Em Pods estáticos, a spec de container é imutável após criação. Para corrigir, delete e recrie.

### Manifest 2 — ResourceQuota excedida

**Sintoma** (`kubectl get pods`):
```
NAME              READY   STATUS    RESTARTS   AGE
app-sem-recursos  0/1     Pending   0          Xm
```

**Causa raiz** (`kubectl describe pod app-sem-recursos -n estudo`):
```
Events:
  Warning  FailedScheduling  Xs  default-scheduler  
    0/3 nodes are available: 3 Insufficient memory. 
    preemption: 0/3 nodes are available: 3 No preemption victims found for incoming pod.
```

Mas a causa real está na ResourceQuota:
```bash
sudo kubectl describe resourcequota quota-apertada -n estudo
# limits.memory: 1Mi usado/1Mi total → sem espaço para 128Mi
```

**Correção:**
```bash
sudo kubectl delete resourcequota quota-apertada -n estudo
# Pod entra em Pending → Running automaticamente após a quota ser removida
```

> [!warning] Importante
> ResourceQuota com `limits.memory: 1Mi` impede **qualquer** Pod que declare `limits.memory > 1Mi`. A quota é verificada no momento do agendamento, não depois. O Pod fica em Pending indefinidamente.

### Manifest 3 — Service sem Endpoints

**Sintoma** (`kubectl get endpoints`):
```
NAME               ENDPOINTS   AGE
svc-sem-endpoints  <none>      Xm
```

**Causa raiz** (`kubectl describe svc svc-sem-endpoints -n estudo`):
```
Selector: app=nao-existe-no-cluster
```

Nenhum Pod no namespace tem `app=nao-existe-no-cluster`.

**Correção:**
```bash
sudo kubectl run pod-para-svc \
  --image=nginx:1.27 \
  --labels="app=nao-existe-no-cluster" \
  -n estudo
```
Ou corrigir o selector do Service para apontar para Pods existentes.

### Manifest 4 — CrashLoopBackOff

**Sintoma** (`kubectl get pods`):
```
NAME        READY   STATUS             RESTARTS   AGE
app-crash   0/1     CrashLoopBackOff   4          Xm
```

**Causa raiz** (`kubectl logs app-crash -n estudo`):
```
# Sem output — o container saiu imediatamente com exit code 1
```

**`kubectl describe`:**
```
Last State: Terminated
  Exit Code: 1
  Reason:    Error
```

**Correção** — mudar o command para algo que não saia com erro:
```bash
sudo kubectl delete pod app-crash -n estudo
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-crash
  namespace: estudo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c", "sleep 3600"]
```

> [!tip] Padrão CrashLoopBackOff
> O backoff é exponencial: 10s → 20s → 40s → 80s → ... até 5 minutos. Para ver o log de uma iteração anterior:
> ```bash
> kubectl logs app-crash -n estudo --previous
> ```
> `--previous` é o flag mais útil de troubleshooting — mostra o que o container disse antes de morrer.

---

## Exercício 9 — Multi-container Pods

> [!warning] Bugs encontrados ao aplicar no homelab
> Os três manifests tinham problemas reais que impediam funcionamento. Os manifests abaixo estão corrigidos e validados para K8s v1.35.3 + busybox:1.36 + nginx:1.27.

---

### Parte A — Sidecar

#### Por que o manifest original falha

O manifest original usava `command: ["/bin/sh", "-c", "tail -f /var/log/nginx/access.log"]` no log-reader. Dois problemas simultâneos:

**Problema 1 — race condition:** Os dois containers de um Pod iniciam em paralelo. Se o container `log-reader` inicia antes do nginx criar o arquivo `access.log`, `tail -f` (minúsculo) falha imediatamente com:
```
tail: can't open '/var/log/nginx/access.log': No such file or directory
```
O container sai com exit code 1 → `CrashLoopBackOff` → Pod fica `1/2` e nunca chega a `2/2`.

**Problema 2 — nginx:1.27 não tem curl:** O comando de verificação original era `kubectl exec -c app -- curl -s localhost`. A imagem `nginx:1.27` é baseada em `debian:bookworm-slim` e **não inclui curl nem wget**. O exec falha com "command not found".

**Problema 3 — emptyDir apaga os symlinks do nginx:** A imagem oficial do nginx cria `/var/log/nginx/access.log` como symlink para `/dev/stdout`. Montar um `emptyDir` sobre `/var/log/nginx` substitui o diretório — os symlinks somem. O nginx então **cria um arquivo real** no emptyDir (comportamento de fallback), o que é na verdade o comportamento que queremos. Isso não é um bug, mas explica por que o sidecar consegue ler o arquivo.

#### Manifest corrigido

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-sidecar
  namespace: estudo
spec:
  containers:
  - name: app
    image: nginx:1.27
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  - name: log-reader
    image: busybox:1.36
    command:
    - /bin/sh
    - -c
    - |
      until [ -f /var/log/nginx/access.log ]; do
        echo "aguardando nginx criar access.log..."
        sleep 1
      done
      exec tail -f /var/log/nginx/access.log
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  volumes:
  - name: logs
    emptyDir: {}
```

**O que mudou:** o `log-reader` aguarda o arquivo existir antes de fazer o `tail`. O `exec` substitui o shell pelo `tail` (boa prática — PID 1 fica sendo o processo relevante).

#### Verificação correta

```bash
# Confirmar READY 2/2
sudo kubectl get pod app-sidecar -n estudo
# NAME          READY   STATUS    RESTARTS   AGE
# app-sidecar   2/2     Running   0          Xm

# Gerar log: usar log-reader (busybox tem wget, compartilha rede com nginx)
sudo kubectl exec -n estudo app-sidecar -c log-reader -- wget -qO- http://localhost/

# Ver log pelo sidecar — deve aparecer a entrada de acesso
sudo kubectl logs app-sidecar -n estudo -c log-reader
# 127.0.0.1 - - [DD/Mon/YYYY:HH:MM:SS +0000] "GET / HTTP/1.1" 200 615 "-" "Wget" "-"
```

> [!info] Por que `wget` no log-reader e não no nginx?
> Containers de um mesmo Pod **compartilham o network namespace** — mesma interface, mesmo IP, mesmas portas. O log-reader (busybox) acessa `localhost:80` que é o nginx rodando no mesmo Pod. Não é comunicação entre containers — é comunicação dentro do mesmo namespace de rede, como dois processos no mesmo host.

> [!tip] emptyDir e persistência
> `emptyDir` existe enquanto o Pod existe. Deletou o Pod → logs perdidos. Para logs persistentes no homelab: use PVC com `nfs-homelab` (retain) ou `nfs-homelab-delete` (delete).

---

### Parte B — Init Container

#### Por que o manifest original pode falhar

`nc -z webapp-svc 80` sem timeout: quando `webapp-svc` não existe, o DNS retorna NXDOMAIN e `nc` sai com erro imediatamente — o loop funciona. O problema potencial é quando `nc` tenta conexão TCP sem timeout e trava aguardando. A flag `-w 2` (timeout de 2 segundos) garante que `nc` nunca trava.

#### Manifest corrigido

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-init
  namespace: estudo
spec:
  initContainers:
  - name: aguarda-servico
    image: busybox:1.36
    command:
    - /bin/sh
    - -c
    - |
      until nc -z -w 2 webapp-svc 80 2>/dev/null; do
        echo "aguardando webapp-svc..."
        sleep 2
      done
      echo "webapp-svc disponivel!"
  containers:
  - name: app
    image: nginx:1.27
```

**O que mudou:** adicionado `-w 2` (timeout de 2s por tentativa) e `2>/dev/null` (suprime erros de DNS para output mais limpo).

#### Verificação correta

```bash
# Antes do Service — Pod fica em Init:0/1
sudo kubectl get pod app-init -n estudo
# NAME       READY   STATUS     RESTARTS   AGE
# app-init   0/1     Init:0/1   0          Xm

# Ver o loop em ação
sudo kubectl logs app-init -n estudo -c aguarda-servico
# aguardando webapp-svc...
# aguardando webapp-svc...
# ...

# Criar o Service (pode ser para qualquer Pod Running no namespace estudo)
sudo kubectl expose pod <nome-pod-existente> \
  --name=webapp-svc \
  --port=80 \
  -n estudo

# Pod avança para Running
sudo kubectl get pod app-init -n estudo -w
# app-init   0/1   Init:0/1     0    Xm
# app-init   0/1   PodInitializing   0    Xm
# app-init   1/1   Running           0    Xm
```

> [!info] `Init:0/1` vs `PodInitializing`
> `Init:0/1` = init containers ainda rodando (0 de 1 completaram).
> `PodInitializing` = todos os init containers terminaram (exit 0), containers principais iniciando.
> O Pod principal só sobe **depois** que todos os init containers terminam com sucesso.

---

### Parte C — Ephemeral Container

#### Por que o manifest original falha

**Problema principal:** `gcr.io/distroless/static:nonroot` é uma imagem verdadeiramente mínima — contém apenas certificados CA, timezone data e `/etc/passwd`. **Não tem nenhum executável**, incluindo `/bin/true`. O container falha para iniciar:
```
OCI runtime create failed: unable to start container process:
exec: "/bin/true": stat /bin/true: no such file or directory
```
Status no kubectl: `Error` ou `ContainerCannotRun`.

**Problema secundário:** mesmo que o container iniciasse, `restartPolicy: Never` com um processo que sai imediatamente coloca o Pod em `Completed`. Um ephemeral container interativo não funciona em Pod Completed — o shell sai imediatamente porque não há processo vivo para se acoplar.

**Solução:** usar `registry.k8s.io/pause:3.9` — a imagem que o próprio Kubernetes usa para o container de infraestrutura de cada Pod. Ela:
- Contém apenas um binário: `/pause`
- Não tem shell, não tem ferramentas
- Roda indefinidamente (nunca sai) → Pod sempre `Running`
- Já está em cache em todos os nodes do cluster (usada internamente pelo kubelet)

#### Manifest corrigido

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-sem-shell
  namespace: estudo
spec:
  containers:
  - name: app-sem-shell
    image: registry.k8s.io/pause:3.9
```

Sem `command` override — o pause usa seu próprio entrypoint (`/pause`).
Sem `restartPolicy: Never` — o pause nunca sai, Pod fica `Running` indefinidamente.

#### Verificação correta

```bash
# Pod deve estar Running
sudo kubectl get pod app-sem-shell -n estudo
# NAME            READY   STATUS    RESTARTS   AGE
# app-sem-shell   1/1     Running   0          Xm

# Tentar exec — deve falhar (sem shell)
sudo kubectl exec -it app-sem-shell -n estudo -- /bin/sh
# OCI runtime exec failed: exec: "/bin/sh": stat /bin/sh: no such file or directory: unknown

# Adicionar ephemeral container de debug
sudo kubectl debug -it app-sem-shell \
  -n estudo \
  --image=busybox:1.36 \
  --target=app-sem-shell

# Dentro do ephemeral container:
/ # ps aux
# PID   USER     TIME  COMMAND
#     1 root      0:00 /pause         ← processo do container principal
#     7 root      0:00 sh             ← shell do ephemeral container
#    13 root      0:00 ps aux

# Inspecionar o processo /pause via /proc
/ # ls /proc/1/
# cmdline  environ  exe  fd  maps  ...

/ # cat /proc/1/cmdline
# /pause

# Sair do ephemeral container
/ # exit

# Verificar que ephemeral container está registrado na spec
sudo kubectl get pod app-sem-shell \
  -n estudo \
  -o jsonpath='{.spec.ephemeralContainers[0].name}'
# debugger  (nome gerado pelo kubectl debug)

# Output completo formatado
sudo kubectl get pod app-sem-shell \
  -n estudo \
  -o jsonpath='{.spec.ephemeralContainers}' | python3 -m json.tool
```

> [!info] Por que `ps aux` mostra o processo do pause?
> A flag `--target=app-sem-shell` compartilha o **PID namespace** do container `app-sem-shell` com o ephemeral container. Normalmente cada container tem PID namespace isolado (PID 1 = processo principal do container). Com `--target`, o ephemeral container entra no mesmo namespace e enxerga todos os processos do target. Isso permite usar `strace -p 1`, `cat /proc/1/environ`, `ls /proc/1/fd` para inspecionar o processo principal sem ter shell no container original.

> [!warning] Ephemeral containers são imutáveis e não removíveis
> Uma vez adicionado com `kubectl debug`, o ephemeral container fica registrado na spec permanentemente até o Pod ser deletado. Não é possível removê-lo. Se precisar de outro ephemeral container no mesmo Pod, adicione com nome diferente — o kubectl debug gera nomes sequenciais automaticamente (`debugger`, `debugger-2`, etc.).

> [!tip] Quando usar cada padrão em produção
> | Padrão | Quando usar |
> |---|---|
> | **Sidecar** | Log shipping, proxy, metrics exporter — processo auxiliar de longa duração |
> | **Init Container** | Migrations de banco, aguardar dependências, setup de configuração — executado uma vez antes da app |
> | **Ephemeral Container** | Debug de container sem shell, diagnóstico de CrashLoopBackOff, inspeção via /proc |

---

## Resumo de Conceitos Verificados

| Exercício | Conceito central | Pergunta-teste |
|---|---|---|
| 1 | Observabilidade passiva | Você consegue mapear o cluster sem tocar em nada? |
| 2 | Efemeridade do Pod | Por que o IP muda? O que resolve isso? |
| 3 | Self-healing e QoS | Quanto tempo leva? Quais Pods são removidos no scale-down? |
| 4 | Service e DNS | Por que ClusterIP é estável? Como o Endpoint é atualizado? |
| 5 | RollingUpdate e rollback | Por que os Pods antigos sobrevivem durante falha? |
| 6 | Labels como cola | Como Services encontram Pods? O que é um mismatch? |
| 7 | kubectl explain offline | Você consegue construir YAML sem internet? |
| 8 | Diagnóstico sistemático | Qual é o fluxo get → describe → logs → events? |
| 9 | Multi-container patterns | Quando usar sidecar vs init vs ephemeral? |

---

## Limpeza final

```bash
sudo kubectl delete namespace estudo
# Remove tudo: Pods, Deployments, Services, ResourceQuotas, etc.
```

---

> [!info] Próximo passo
> [[11-Exercicios/configmaps]] — Externalizar configuração com ConfigMaps. Formas de criação (`create configmap`, `--from-file`, `--from-env-file`) e formas de consumo (`env`, `envFrom`, volume). Comportamento de update: volumes atualizam automaticamente, variáveis de ambiente **não**.
