---
tags:
  - kubernetes
  - jobs
  - cronjobs
area: kubernetes
tipo: conteudo
prerequisites:
  - "[[05-Kubernetes/kubernetes-teoria-inicial]]"
  - "[[05-Kubernetes/labels-annotations]]"
next:
  - "[[11-Exercicios/jobs-cronjobs]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Jobs e CronJobs

> Exercícios: [[11-Exercicios/jobs-cronjobs]] | Trilha: [[00-Trilha/kubernetes]]

---

## O que é e por que existe

**Deployment** garante que N réplicas estejam sempre rodando. Se o Pod termina, ele reinicia. Isso é exatamente o comportamento **errado** para tarefas batch: backups, migrations, exportação de relatórios, processamento de filas — trabalhos que devem rodar até completar e então parar.

**Job** resolve isso: cria Pods que executam até terminar com sucesso (`exit code 0`), e não os reinicia quando terminam. O Job garante que a tarefa foi completada com sucesso o número de vezes especificado — se o Pod falhar, o Job cria um novo.

**CronJob** adiciona scheduling ao Job: "execute este Job toda sexta às 23h". É o cron do sistema operacional, mas com todas as garantias de resiliência do Kubernetes.

A distinção fundamental:

| Workload | Termina? | Reinicia se terminar? | Reinicia se falhar? |
|---|---|---|---|
| Deployment | Nunca (idealmente) | Sim | Sim |
| Job | Sim (quando completa) | Não | Sim (até backoffLimit) |
| CronJob | Sim (via Job) | Não | Sim (via Job) |

---

## Como funciona internamente

### Job Controller

O Job controller (dentro do controller-manager) observa objetos Job e gerencia o ciclo de vida dos Pods:

```
Job criado
  └─ Job controller cria Pod(s) conforme parallelism
       ├─ Pod termina com exit 0 → incrementa succeeded counter
       │    └─ succeeded == completions → Job marcado como Complete
       └─ Pod termina com exit ≠ 0
            ├─ restartPolicy: OnFailure → kubelet reinicia o container no mesmo Pod
            └─ restartPolicy: Never → Job controller cria novo Pod
                 └─ falhas acumuladas > backoffLimit → Job marcado como Failed
```

O Job controller usa **exponential backoff** entre tentativas de re-criar Pods: 10s, 20s, 40s... até 6 minutos (cap). Isso evita flood de Pods falhando rapidamente.

### CronJob Controller

O CronJob controller roda um loop a cada **10 segundos** — não é um timer real de cron. Ele verifica:
1. Qual o próximo horário agendado com base no campo `schedule`
2. Se é hora de criar um Job (ou se perdeu janelas — missed schedules)
3. Se deve criar ou não com base no `concurrencyPolicy`

```
CronJob controller (loop a cada 10s)
  ├─ Calcular próximo horário de disparo
  ├─ Verificar schedules perdidos (missed)
  │    └─ Se > 100 schedules perdidos → para de criar Jobs, emite warning
  ├─ Verificar concurrencyPolicy
  │    ├─ Allow → criar Job mesmo se anterior ainda roda
  │    ├─ Forbid → pular se Job anterior ainda existe
  │    └─ Replace → deletar Job anterior e criar novo
  └─ Criar Job a partir do jobTemplate
```

### restartPolicy: Never vs OnFailure

Esta é uma das escolhas mais impactantes em Jobs:

**`Never`** — quando o container falha, o Pod vai para `Failed` e o Job cria um **novo Pod**. Resultado: você acumula Pods Failed no histórico. Mais Pods criados, mas cada falha fica registrada com seus próprios logs.

**`OnFailure`** — quando o container falha, o kubelet reinicia o container **no mesmo Pod** (sem criar novo Pod). O Pod continua `Running`, apenas o container é reiniciado. Mais eficiente em recursos, mas os logs da tentativa anterior são perdidos quando o container reinicia.

Regra prática: `Never` para debugging (logs preservados), `OnFailure` para produção quando a tarefa é idempotente e logs de falhas anteriores não são críticos.

### Contagem de Completions

O Job rastreia `status.succeeded` e `status.failed`. Um Job completa quando `succeeded >= spec.completions`. Se `completions` não for especificado, o Job completa quando o primeiro Pod termina com sucesso (`completions=1` implícito).

```yaml
status:
  startTime: "2026-04-18T02:00:00Z"
  completionTime: "2026-04-18T02:05:32Z"
  succeeded: 3        # número de Pods que terminaram com exit 0
  failed: 1           # número de Pods que falharam (não conta tentativas OnFailure)
  active: 0           # Pods atualmente rodando
  conditions:
  - type: Complete
    status: "True"
```

---

## Spec Completa — Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: exportar-relatorio
  namespace: estudo
spec:
  # Quantas vezes a tarefa deve completar com sucesso
  completions: 1

  # Quantos Pods podem rodar em paralelo
  parallelism: 1

  # Tentativas máximas antes de o Job ser marcado como Failed
  backoffLimit: 3

  # Tempo máximo de vida do Job (segundos) — mata Pods ativos se exceder
  activeDeadlineSeconds: 3600

  # Apagar o Job automaticamente N segundos após completar/falhar
  ttlSecondsAfterFinished: 86400  # 24 horas

  # Indexed ou NonIndexed (ver seção Parallel Jobs)
  completionMode: NonIndexed

  template:
    spec:
      restartPolicy: Never    # Never ou OnFailure — Deployment não aceita esses valores
      containers:
      - name: exportador
        image: python:3.12-slim
        command: ["python", "/app/export.py"]
        env:
        - name: OUTPUT_DIR
          value: "/data/exports"
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            memory: "512Mi"
```

---

## Spec Completa — CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-banco
  namespace: estudo
spec:
  # Sintaxe cron padrão: minuto hora dia-do-mês mês dia-da-semana
  schedule: "0 3 * * *"      # toda noite às 03:00

  # Fuso horário (Kubernetes >= 1.27)
  timeZone: "America/Sao_Paulo"

  # O que fazer se o Job anterior ainda está rodando quando este dispara
  # Allow (padrão) | Forbid | Replace
  concurrencyPolicy: Forbid

  # Se o CronJob perder o horário agendado por mais de N segundos,
  # considera a execução como "missed" e não tenta recuperar
  startingDeadlineSeconds: 300   # 5 minutos de tolerância

  # Quantos Jobs bem-sucedidos manter no histórico
  successfulJobsHistoryLimit: 3

  # Quantos Jobs com falha manter no histórico
  failedJobsHistoryLimit: 5

  # Suspender temporariamente sem deletar (false = ativo)
  suspend: false

  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 7200    # 2 horas máximo
      ttlSecondsAfterFinished: 604800 # 7 dias
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres:16-alpine
            command:
            - /bin/sh
            - -c
            - |
              pg_dump -h $DB_HOST -U $DB_USER $DB_NAME | \
              gzip > /backup/$(date +%Y%m%d_%H%M%S).sql.gz
            env:
            - name: DB_HOST
              value: "postgres.databases.svc.cluster.local"
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

---

## Parallel Jobs — Padrões

### 1. Job Único (padrão)

```yaml
completions: 1
parallelism: 1
```
Um Pod, uma execução. O caso mais simples.

### 2. Fixed Completion Count (sem índice)

```yaml
completions: 5
parallelism: 2
```
5 execuções independentes, 2 por vez. O Job não sabe qual é qual — cada Pod recebe a mesma tarefa. Útil quando a tarefa é stateless e você quer executar N vezes (ex: processar N arquivos onde o código seleciona o próximo disponível de uma fila).

### 3. Indexed Jobs (completionMode: Indexed)

Introduzido no Kubernetes 1.24. Cada Pod recebe um **índice único** (0 a completions-1) via:
- Variável de ambiente `JOB_COMPLETION_INDEX`
- Hostname do Pod: `<job-name>-<index>`
- Arquivo `/etc/pod-info/JOB_COMPLETION_INDEX`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: processar-chunks
  namespace: estudo
spec:
  completions: 10        # processará chunks 0 a 9
  parallelism: 3         # 3 Pods por vez
  completionMode: Indexed
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: python:3.12-slim
        command:
        - python
        - -c
        - |
          import os
          chunk_index = int(os.environ['JOB_COMPLETION_INDEX'])
          print(f"Processando chunk {chunk_index} de 10")
          # processar arquivo chunk_{chunk_index}.csv
```

O Job garante que cada índice será completado exatamente uma vez, mesmo com falhas e recriacões de Pod.

### 4. Parallel Work Queue (sem completions fixas)

```yaml
completions: null    # omitido — Job termina quando qualquer Pod conclui com sucesso
parallelism: 5
backoffLimit: 50
```
Útil para work queues (SQS, RabbitMQ, Redis): os Pods puxam itens da fila e o Job termina quando a fila está vazia (o primeiro Pod a encontrar fila vazia retorna exit 0, terminando o Job).

---

## Na prática — Comandos e Exemplos Reais

### Gerenciar Jobs

```bash
# Criar Job manualmente
kubectl create job teste --image=busybox -n estudo -- /bin/sh -c "echo hello && sleep 5"

# Listar Jobs com status
kubectl get jobs -n estudo
# NAME    COMPLETIONS   DURATION   AGE
# teste   1/1           7s         15s

# Ver Pods gerados pelo Job
kubectl get pods -n estudo -l job-name=teste

# Logs do Job (Pod gerado)
kubectl logs -n estudo -l job-name=teste

# Deletar Job (deleta também os Pods gerados)
kubectl delete job teste -n estudo

# Forçar execução de um CronJob imediatamente (criar Job manualmente do template)
kubectl create job --from=cronjob/backup-banco teste-manual -n estudo
```

### Inspecionar Status

```bash
# Status detalhado do Job
kubectl describe job backup-banco -n estudo

# Ver a condição de Complete/Failed
kubectl get job backup-banco -n estudo -o jsonpath='{.status.conditions[*].type}'

# Watch em tempo real
kubectl get pods -n estudo -l job-name=backup-banco -w

# Ver histórico de Jobs criados por um CronJob
kubectl get jobs -n estudo --sort-by='.status.startTime'
```

### Suspend e Resume

```bash
# Suspender CronJob (sem deletar — novo Jobs não serão criados)
kubectl patch cronjob backup-banco -n estudo -p '{"spec":{"suspend":true}}'

# Retomar
kubectl patch cronjob backup-banco -n estudo -p '{"spec":{"suspend":false}}'

# Suspender um Job em execução (para Pods ativos imediatamente)
kubectl patch job processamento -n estudo -p '{"spec":{"suspend":true}}'
```

### Sintaxe Cron

```
┌─────────── minuto (0-59)
│ ┌───────── hora (0-23)
│ │ ┌─────── dia do mês (1-31)
│ │ │ ┌───── mês (1-12 ou jan-dec)
│ │ │ │ ┌─── dia da semana (0-6, 0=domingo, ou sun-sat)
│ │ │ │ │
* * * * *

Exemplos:
"0 3 * * *"      → 03:00 todo dia
"*/15 * * * *"   → a cada 15 minutos
"0 0 * * 0"      → domingo meia-noite
"0 9-17 * * 1-5" → a cada hora de 09h a 17h, segunda a sexta
"0 0 1 * *"      → 1º de cada mês à meia-noite
"@daily"         → equivalente a "0 0 * * *"
"@hourly"        → equivalente a "0 * * * *"
```

---

## Casos de Uso e Boas Práticas

### Migrations de Banco de Dados

O padrão mais comum de Job em produção:

```yaml
# Job de migration antes do Deployment da nova versão
apiVersion: batch/v1
kind: Job
metadata:
  name: migration-v2-3
  namespace: producao
  annotations:
    kubernetes.io/change-cause: "migration: adiciona coluna user_preferences"
spec:
  backoffLimit: 0       # migrations não são idempotentes — falha uma vez, para
  activeDeadlineSeconds: 300
  ttlSecondsAfterFinished: 86400
  template:
    spec:
      restartPolicy: Never
      serviceAccountName: migration-sa
      containers:
      - name: migration
        image: app:v2.3
        command: ["python", "manage.py", "migrate", "--no-input"]
        envFrom:
        - secretRef:
            name: db-credentials
```

> [!warning] backoffLimit: 0 em migrations
> Migrations que alteram schema raramente são idempotentes. `backoffLimit: 0` garante que o Job falha imediatamente sem tentar novamente — evita aplicar a mesma migration duas vezes e corromper dados.

### Backup com CronJob + PVC Compartilhado

```yaml
# PVC para armazenar backups (nfs-homelab para persistência)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backup-storage
  namespace: estudo
spec:
  accessModes: [ReadWriteMany]   # múltiplos Pods podem montar simultaneamente
  storageClassName: nfs-homelab
  resources:
    requests:
      storage: 50Gi
```

### Limpeza de Recursos Antigos (TTL)

Sem `ttlSecondsAfterFinished`, Jobs concluídos ficam para sempre no cluster — acumulando Pods em estado `Completed` que ocupam espaço no etcd e poluem os logs:

```bash
# Ver Jobs acumulados (sem TTL)
kubectl get jobs -A | wc -l
# 847   ← 847 Jobs históricos em um cluster de 1 ano sem TTL

# Limpeza manual de todos os Jobs concluídos
kubectl delete jobs --field-selector=status.successful=1 -A
```

O TTL Controller deleta Job e todos os seus Pods automaticamente N segundos após `completionTime`.

### Idempotência é Obrigatória

Jobs podem re-executar o mesmo código se o Pod falhar. O código do Job **deve ser idempotente** — executar duas vezes deve ter o mesmo resultado que executar uma:

- Backup: sempre cria arquivo com timestamp único → idempotente
- `INSERT`: usar `INSERT ... ON CONFLICT DO NOTHING` → idempotente
- Processar item de fila: marcar como `processing` antes de processar, checar antes de processar → idempotente
- `DROP TABLE IF EXISTS; CREATE TABLE`: idempotente
- `ALTER TABLE ADD COLUMN` sem `IF NOT EXISTS`: **não idempotente** — falha na segunda execução

### concurrencyPolicy — Evitar Sobreposição

Para CronJobs onde a execução pode durar mais que o intervalo:

```yaml
concurrencyPolicy: Forbid    # recomendado para backups e migrations
```

Com `Forbid`, se o Job das 03:00 ainda estiver rodando quando as 04:00 chegarem, o horário das 04:00 é pulado (e registrado no `status.lastMissedScheduleTime`).

Com `Allow` (padrão), dois Jobs rodam simultaneamente — problema para backups que escrevem no mesmo destino.

---

## Troubleshooting — Cenários Reais de Produção

### Job stuck em "0/1 completions"

```bash
kubectl describe job <job-name> -n <ns>
# Events: geralmente revelam o problema

kubectl get pods -n <ns> -l job-name=<job-name>
# Pod em CrashLoopBackOff, Pending, ou ImagePullBackOff?

kubectl logs -n <ns> <pod-name> --previous   # logs da tentativa anterior
```

Causas comuns:
- `backoffLimit` atingido — Job para de criar Pods; status `Failed`
- Pod em `Pending` por falta de recursos ou node selector impossível
- `ImagePullBackOff` — imagem não existe ou sem credenciais
- `OOMKilled` — limite de memória muito baixo para a tarefa

### CronJob não dispara no horário esperado

```bash
kubectl describe cronjob <nome> -n <ns>
# Procurar: "Last Schedule Time", "Active Jobs", e Events

# Verificar se o CronJob está suspenso
kubectl get cronjob <nome> -n <ns> -o jsonpath='{.spec.suspend}'

# Verificar missed schedules
kubectl get cronjob <nome> -n <ns> -o jsonpath='{.status.lastMissedScheduleTime}'
```

Causas comuns:
1. **`startingDeadlineSeconds` muito pequeno**: se o controller-manager ficou offline por mais tempo que o deadline, o CronJob considera os schedules perdidos e não tenta recuperar
2. **Mais de 100 missed schedules**: o controller para de criar Jobs — precisa recriar o CronJob
3. **`concurrencyPolicy: Forbid` com Job longo**: schedule pulado porque Job anterior ainda está rodando
4. **`suspend: true`**: CronJob suspenso explicitamente

### backoffLimit atingido — Job Failed

```bash
kubectl get job <nome> -n <ns> -o jsonpath='{.status.conditions[?(@.type=="Failed")].message}'
# "Job has reached the specified backoff limit"

# Ver todos os Pods que foram criados (incluindo os failed)
kubectl get pods -n <ns> -l job-name=<nome> --show-all
# ou com --field-selector
kubectl get pods -n <ns> -l job-name=<nome> --field-selector=status.phase=Failed

# Logs do último Pod que falhou
kubectl logs -n <ns> $(kubectl get pods -n <ns> -l job-name=<nome> --sort-by='.metadata.creationTimestamp' -o name | tail -1)
```

### activeDeadlineSeconds expirando

```bash
kubectl describe job <nome> -n <ns>
# Condition: type=Failed reason=DeadlineExceeded
# Events: Job was active longer than specified deadline

# O activeDeadlineSeconds conta desde o startTime do Job, não do Pod
# Para jobs longos, aumentar o deadline ou dividir em Jobs menores
```

### Pods `Completed` acumulando (sem TTL)

```bash
# Verificar quantos Pods Completed existem
kubectl get pods -A --field-selector=status.phase=Succeeded | wc -l

# Adicionar TTL a um CronJob existente retroativamente
kubectl patch cronjob <nome> -n <ns> --type=merge \
  -p '{"spec":{"jobTemplate":{"spec":{"ttlSecondsAfterFinished":86400}}}}'

# Limpeza pontual de Pods Completed em um namespace
kubectl delete pods -n <ns> --field-selector=status.phase=Succeeded
```

---

## Nível Avançado — Edge Cases e CKA/Staff

### Pod Failure Policy (Kubernetes 1.26+)

Controle granular sobre o que acontece quando um Pod falha — em vez de simplesmente incrementar o contador de falhas:

```yaml
spec:
  backoffLimit: 6
  podFailurePolicy:
    rules:
    # Se o container saiu com exit code 42 (código de "dados inválidos"),
    # marcar Job como Failed imediatamente sem mais tentativas
    - action: FailJob
      onExitCodes:
        containerName: processador
        operator: In
        values: [42, 43]
    # Se o Pod foi evicted (OOM, node pressure), não contar como falha do Job
    - action: Ignore
      onPodConditions:
      - type: DisruptionTarget
```

Isso evita o problema clássico: `backoffLimit: 6` + bug de lógica = 6 Pods falhando desnecessariamente quando o primeiro já deveria ter sinalizado o erro definitivo.

### Job com Init Containers para Dependências

```yaml
template:
  spec:
    initContainers:
    - name: wait-for-db
      image: busybox
      command:
      - /bin/sh
      - -c
      - |
        until nc -z postgres.databases.svc.cluster.local 5432; do
          echo "Aguardando PostgreSQL..."
          sleep 5
        done
    containers:
    - name: migration
      image: app:latest
      command: ["python", "migrate.py"]
```

### Indexed Jobs com ConfigMap como Work Queue Estático

Para distribuir trabalho de forma determinística sem serviço de fila:

```yaml
# ConfigMap com lista de arquivos a processar
apiVersion: v1
kind: ConfigMap
metadata:
  name: arquivos-para-processar
  namespace: estudo
data:
  items.json: |
    ["arquivo_0.csv","arquivo_1.csv","arquivo_2.csv","arquivo_3.csv","arquivo_4.csv"]
---
apiVersion: batch/v1
kind: Job
metadata:
  name: processar-arquivos
  namespace: estudo
spec:
  completions: 5
  parallelism: 3
  completionMode: Indexed
  template:
    spec:
      restartPolicy: Never
      volumes:
      - name: config
        configMap:
          name: arquivos-para-processar
      containers:
      - name: worker
        image: python:3.12-slim
        volumeMounts:
        - name: config
          mountPath: /config
        command:
        - python
        - -c
        - |
          import json, os
          idx = int(os.environ['JOB_COMPLETION_INDEX'])
          with open('/config/items.json') as f:
              items = json.load(f)
          arquivo = items[idx]
          print(f"Processando {arquivo}")
```

### CronJob com Timezone e Horário de Verão

```yaml
spec:
  schedule: "0 2 * * *"
  timeZone: "America/Sao_Paulo"   # respeita horário de verão automaticamente
```

Antes do Kubernetes 1.27, o timezone do controller-manager era o do node — o que causava bugs silenciosos quando o cluster estava em UTC mas o negócio precisava de 02:00 em São Paulo.

### Job como Parte de Pipeline GitOps

Jobs são frequentemente o mecanismo de `pre-sync` em ArgoCD — executados antes do deploy da aplicação:

```yaml
# ArgoCD hook: roda antes do sync
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

### Monitorar Jobs com Prometheus

O kube-state-metrics expõe métricas de Jobs automaticamente:

```promql
# Jobs com falha atualmente
kube_job_status_failed > 0

# CronJobs sem execução recente (pode indicar suspend acidental ou missed schedules)
time() - kube_cronjob_status_last_schedule_time > 86400  # mais de 24h sem execução

# Duração de Jobs (útil para alertar em jobs muito longos)
kube_job_status_completion_time - kube_job_status_start_time
```

### Parallel Job com Semaphore (limitar concorrência sem mudar parallelism)

Quando `parallelism` não é suficiente para controlar acesso a recursos compartilhados:

```yaml
containers:
- name: worker
  image: redis:7-alpine
  command:
  - /bin/sh
  - -c
  - |
    # Adquirir lock via Redis antes de processar
    LOCK_KEY="job-lock-${JOB_COMPLETION_INDEX}"
    redis-cli -h redis.estudo.svc SET $LOCK_KEY $JOB_COMPLETION_INDEX NX EX 3600
    # Processar somente se lock adquirido
    # ...
    redis-cli -h redis.estudo.svc DEL $LOCK_KEY
```

---

## Referências

- [Jobs — Run to Completion](https://kubernetes.io/docs/concepts/workloads/controllers/job/) — spec completa
- [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) — schedule, concurrencyPolicy
- [Indexed Job pattern](https://kubernetes.io/docs/tasks/job/indexed-parallel-processing-static/) — distribuição determinística
- [Pod Failure Policy](https://kubernetes.io/docs/tasks/job/pod-failure-policy/) — controle granular de falhas
- [Fine Parallel Processing](https://kubernetes.io/docs/tasks/job/fine-parallel-processing-work-queue/) — work queue pattern
- DK8S Day4 — `/mnt/c/Users/felip/Documents/Kubernets/DK8S`
