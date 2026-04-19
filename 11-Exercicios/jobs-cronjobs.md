---
tags:
  - exercicios
  - kubernetes
  - jobs
  - cronjobs
tipo: exercicios
area: kubernetes
conteudo: "[[05-Kubernetes/jobs-cronjobs]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Exercícios — Jobs e CronJobs

> Conteúdo: [[05-Kubernetes/jobs-cronjobs]] | Trilha: [[00-Trilha/kubernetes]]

> [!warning] Restrições de Produção
> Namespaces `controle-gastos`, `promobot` e `databases` → **somente leitura**.
> Nunca criar Jobs ou CronJobs nos namespaces de produção.
> Todos os exercícios usam o namespace `estudo`.

---

## Exercício 1: Inventário de Jobs e CronJobs no Cluster

**Contexto:** Você assumiu um cluster que tem Jobs e CronJobs rodando há meses. Nunca foi documentado o que está agendado ou quais Jobs falharam recentemente.

**Missão:** Fazer um inventário completo do estado atual de Jobs e CronJobs no cluster.

**Requisitos:**
- [ ] Listar todos os CronJobs em todos os namespaces — identificar: schedule, se está suspenso, data da última execução, próxima execução esperada
- [ ] Listar todos os Jobs em todos os namespaces e filtrar os que estão com status `Failed`
- [ ] Para cada Job com falha, identificar: namespace, nome, número de falhas, motivo da falha
- [ ] Verificar se há Pods em estado `Completed` ou `Failed` acumulados por Jobs sem `ttlSecondsAfterFinished`
- [ ] Identificar quais CronJobs não têm `concurrencyPolicy` explícito (usando o padrão `Allow`) e avaliar se isso é um risco

**Verificação:**
```bash
# CronJobs com status
sudo kubectl get cronjob -A -o wide

# Jobs com falha
sudo kubectl get jobs -A --field-selector=status.successful=0

# Pods acumulados de Jobs
sudo kubectl get pods -A --field-selector=status.phase=Succeeded | wc -l
sudo kubectl get pods -A --field-selector=status.phase=Failed | wc -l
```

---

## Exercício 2: Job Básico — restartPolicy: Never vs OnFailure

**Contexto:** Você precisa entender empiricamente a diferença de comportamento entre `restartPolicy: Never` e `restartPolicy: OnFailure` antes de escolher qual usar em um Job de migration de banco.

**Missão:** Criar dois Jobs que falham intencionalmente e observar o comportamento de cada restartPolicy.

**Requisitos:**
- [ ] Criar um Job `falha-never` com `restartPolicy: Never`, `backoffLimit: 3`, e comando que sempre sai com `exit 1`
- [ ] Criar um Job `falha-onfailure` com `restartPolicy: OnFailure`, `backoffLimit: 3`, e mesmo comando
- [ ] Para cada Job, observar: quantos Pods são criados? O que aparece em `kubectl get pods`? Os logs de tentativas anteriores são acessíveis?
- [ ] Verificar o `status.failed` de cada Job e compará-los
- [ ] Responder: qual seria a diferença de comportamento se `backoffLimit: 0` fosse usado?
- [ ] Deletar os dois Jobs ao finalizar

**Verificação:**
```bash
# Ver Pods criados por cada Job
sudo kubectl get pods -n estudo -l job-name=falha-never
sudo kubectl get pods -n estudo -l job-name=falha-onfailure

# Status do Job
sudo kubectl get job falha-never -n estudo -o jsonpath='{.status}'
sudo kubectl get job falha-onfailure -n estudo -o jsonpath='{.status}'
```

> [!tip] Observar em Tempo Real
> Use `kubectl get pods -n estudo -w` em um terminal separado enquanto os Jobs estão rodando para ver a diferença em tempo real.

---

## Exercício 3: Job Paralelo com Indexed Completions

**Contexto:** O time de dados precisa processar 6 arquivos CSV independentes. Cada processamento leva entre 10 e 30 segundos. Para acelerar, devem rodar em paralelo (máximo 3 simultâneos). Cada Pod precisa saber qual arquivo é o seu.

**Missão:** Criar um Job Indexed que distribui trabalho entre Pods usando `JOB_COMPLETION_INDEX`.

**Requisitos:**
- [ ] Criar um Job com:
  - `completions: 6`
  - `parallelism: 3`
  - `completionMode: Indexed`
  - `backoffLimit: 2`
  - `ttlSecondsAfterFinished: 300`
  - Cada Pod deve imprimir: `"Processando arquivo_<index>.csv"` e simular trabalho com `sleep $((RANDOM % 20 + 10))`
- [ ] Observar com `kubectl get pods -n estudo -w` a ordem de criação e conclusão dos Pods
- [ ] Verificar que cada índice (0-5) foi executado exatamente uma vez
- [ ] Identificar o nome dos Pods gerados — qual é o padrão de nomenclatura para Indexed Jobs?
- [ ] Confirmar que o Job não criou dois Pods com o mesmo índice simultaneamente

**Verificação:**
```bash
# Pods do Job com seus índices
sudo kubectl get pods -n estudo -l job-name=processar-arquivos \
  -o custom-columns='NAME:.metadata.name,INDEX:.metadata.annotations.batch\.kubernetes\.io/job-completion-index,STATUS:.status.phase'

# Job concluído com 6 completions
sudo kubectl get job processar-arquivos -n estudo
```

---

## Exercício 4: CronJob de Backup com PVC NFS

**Contexto:** O time quer um backup diário do estado de um diretório de configuração. O backup deve ser gravado no NFS do homelab, e não mais de 7 dias de backups devem ser mantidos. O Job não deve rodar em sobreposição se o anterior ainda estiver ativo.

**Missão:** Criar um CronJob completo de backup com PVC NFS, TTL, e concurrencyPolicy correto.

**Requisitos:**
- [ ] Criar um PVC no namespace `estudo` usando a StorageClass `nfs-homelab` com acesso `ReadWriteMany`
- [ ] Criar um CronJob com:
  - Schedule a cada 2 minutos (`*/2 * * * *`) — para testar sem esperar muito
  - `concurrencyPolicy: Forbid`
  - `successfulJobsHistoryLimit: 3` e `failedJobsHistoryLimit: 2`
  - `ttlSecondsAfterFinished: 300`
  - Job que cria um arquivo no PVC com o nome `backup-$(date +%Y%m%d_%H%M%S).txt` contendo `hostname` e `date`
- [ ] Aguardar pelo menos 3 execuções e verificar:
  - Que os arquivos de backup estão sendo criados no PVC (usando um Pod auxiliar que monta o mesmo PVC)
  - Que Jobs antigos estão sendo limpos pelo `successfulJobsHistoryLimit`
- [ ] Suspender o CronJob ao finalizar

**Verificação:**
```bash
# Ver execuções do CronJob
sudo kubectl get jobs -n estudo --sort-by='.metadata.creationTimestamp'

# Verificar arquivos criados no PVC (via Pod auxiliar)
sudo kubectl run verificar --image=busybox -n estudo --restart=Never \
  --overrides='{"spec":{"volumes":[{"name":"bkp","persistentVolumeClaim":{"claimName":"backup-pvc"}}],"containers":[{"name":"v","image":"busybox","command":["ls","-la","/backup"],"volumeMounts":[{"name":"bkp","mountPath":"/backup"}]}]}}' \
  -- ls -la /backup
```

---

## Exercício 5: Debugar CronJob que Não Dispara

**Contexto:** Um colega criou um CronJob há 2 horas e ele nunca disparou. Você precisa diagnosticar o problema.

**Missão:** Reproduzir o cenário com um CronJob quebrado e diagnosticar a causa usando apenas `kubectl`.

**Requisitos:**
- [ ] Criar o seguinte CronJob propositalmente quebrado:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-quebrado
  namespace: estudo
spec:
  schedule: "*/1 * * * *"
  startingDeadlineSeconds: 5
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      activeDeadlineSeconds: 60
      backoffLimit: 1
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: tarefa
            image: busybox
            command: ["sleep", "120"]
```
- [ ] Aguardar 3 minutos e verificar que o CronJob nunca executou (ou executou apenas uma vez e parou)
- [ ] Usando apenas `kubectl describe cronjob cronjob-quebrado -n estudo` e `kubectl get jobs -n estudo`, identificar **todos os problemas** nesse CronJob — deve encontrar pelo menos 3
- [ ] Corrigir os problemas e verificar que o CronJob passa a executar regularmente
- [ ] Deletar ao finalizar

**Verificação:**
```bash
sudo kubectl describe cronjob cronjob-quebrado -n estudo
sudo kubectl get jobs -n estudo -l app=cronjob-quebrado
```

> [!tip] Pista
> Pense em: o que acontece quando `activeDeadlineSeconds` é menor que a duração real do Job? O que `startingDeadlineSeconds: 5` significa em relação ao loop de 10s do CronJob controller? E `concurrencyPolicy: Forbid` com um Job que nunca termina?

---

## Exercício 6: Job de Migration com backoffLimit: 0

**Contexto:** O time vai fazer deploy da v2.0 de uma aplicação que requer uma migration de banco de dados. A migration não é idempotente — se rodar duas vezes, vai corromper dados. Você precisa criar um Job que falha exatamente uma vez e não tenta mais.

**Missão:** Criar e executar um Job de migration seguro com `backoffLimit: 0`.

**Requisitos:**
- [ ] Criar um ConfigMap `migration-script` com o seguinte conteúdo simulado:
```bash
#!/bin/sh
echo "Iniciando migration v2.0..."
echo "ALTER TABLE users ADD COLUMN preferences JSONB DEFAULT '{}';"
# Simular falha na primeira tentativa (arquivo de lock)
if [ ! -f /tmp/migration-done ]; then
  echo "ERRO: Banco não está acessível" >&2
  exit 1
fi
echo "Migration concluída com sucesso"
```
- [ ] Criar um Job que monta o ConfigMap como script e executa com `backoffLimit: 0`
- [ ] Verificar que após a falha o Job está em estado `Failed` e não tenta novamente
- [ ] Verificar os logs do Pod para ver a mensagem de erro
- [ ] Editar o script no ConfigMap para que não falhe (criar o arquivo `/tmp/migration-done` antes da verificação), deletar o Job Failed, e criar um novo Job — confirmar que completa com sucesso

**Verificação:**
```bash
sudo kubectl get job migration-v2 -n estudo
sudo kubectl describe job migration-v2 -n estudo
sudo kubectl logs -n estudo -l job-name=migration-v2
```

> [!warning] Migrations em Produção Real
> Em produção, nunca criar o Job de migration com `kubectl apply` junto com o Deployment no mesmo arquivo — use ferramentas como Helm hooks, ArgoCD PreSync hooks, ou pipelines CI/CD que garantem ordem de execução.

---

## Exercício 7: Monitorar Jobs com Prometheus (kube-state-metrics)

**Contexto:** O time de SRE quer alertar quando qualquer Job falhar no cluster. Você precisa descobrir quais métricas o kube-state-metrics expõe para Jobs e criar uma query PromQL útil.

**Missão:** Explorar as métricas de Jobs no Prometheus e criar queries para monitoramento.

**Requisitos:**
- [ ] No Prometheus UI (`prometheus.staypuff.info`), explorar as métricas com prefixo `kube_job_` — listar todas as métricas disponíveis e o que cada uma representa
- [ ] Criar e testar no Prometheus UI as seguintes queries:
  1. Jobs com falha no momento: `kube_job_status_failed > 0`
  2. Jobs que levaram mais de 5 minutos para completar
  3. CronJobs sem execução nas últimas 25 horas (pode indicar suspend acidental)
  4. Número total de Jobs por namespace
- [ ] Criar um Job que falhe intencionalmente no namespace `estudo` e confirmar que a query (1) o detecta
- [ ] Verificar se existe um alerta pré-configurado no kube-prometheus-stack para Jobs falhados (inspecionar as PrometheusRules existentes)

**Verificação:**
```bash
# No Prometheus UI:
# Targets: verificar se kube-state-metrics está UP
# Graph: testar as queries acima

# Verificar PrometheusRules existentes para Jobs
sudo kubectl get prometheusrule -A -o yaml | grep -A5 'job.*fail\|KubeJob'
```

---

## Exercício 8: Pipeline de Processamento com Indexed Job e Work Queue (Staff-level)

**Contexto:** O time de dados precisa processar um conjunto de 10 relatórios mensais. Cada relatório é independente, o processamento leva tempo variável, e se um falhar, deve ser possível re-executar apenas aquele índice sem reprocessar todos os outros. O resultado de cada relatório deve ser persistido no NFS.

**Missão:** Criar um pipeline completo de processamento usando Indexed Job com persistência de resultados e rastreabilidade.

**Requisitos:**

**Parte A — Infraestrutura:**
- [ ] Criar um ConfigMap `relatorios-config` com a lista de 10 relatórios simulados (em JSON ou texto linha por linha)
- [ ] Criar um PVC `resultados-relatorios` usando `nfs-homelab` com `ReadWriteMany`
- [ ] Criar um ConfigMap `script-processamento` com um script que:
  - Lê o índice via `JOB_COMPLETION_INDEX`
  - Busca o nome do relatório no ConfigMap
  - "Processa" (simula com sleep aleatório entre 5-30s)
  - Grava o resultado em `/results/relatorio_<index>_<nome>.txt` com timestamp
  - Simula falha (exit 1) se o índice for **3** na primeira tentativa (usando arquivo de flag)

**Parte B — Job:**
- [ ] Criar um Indexed Job com:
  - `completions: 10`, `parallelism: 4`, `completionMode: Indexed`
  - `backoffLimit: 3` (para que o índice 3 seja re-tentado e complete na segunda vez)
  - `ttlSecondsAfterFinished: 600`
  - Montando ambos os ConfigMaps e o PVC

**Parte C — Observação e Validação:**
- [ ] Acompanhar o Job em tempo real: `kubectl get pods -n estudo -w`
- [ ] Identificar quando o Pod do índice 3 falha e quando é re-criado pelo Job controller
- [ ] Após completar, verificar que 10 arquivos de resultado existem no PVC
- [ ] Verificar que o Job marcou `succeeded: 10` (mesmo que o índice 3 tenha falhado uma vez)
- [ ] Documentar: qual seria a abordagem se o índice 3 falhasse permanentemente (backoffLimit esgotado)? Como re-executar apenas aquele índice sem recriar o Job inteiro?

**Verificação:**
```bash
# Status do Job
sudo kubectl get job processar-relatorios -n estudo -o jsonpath='{.status}'

# Resultados no PVC (via Pod auxiliar)
sudo kubectl run check-results --image=busybox -n estudo --restart=Never \
  -- ls -la /results

# Confirmar que o índice 3 tem um resultado (foi re-tentado com sucesso)
sudo kubectl run check-3 --image=busybox -n estudo --restart=Never \
  -- cat /results/relatorio_3_*.txt
```

> [!info] Re-executar um índice específico
> Não é possível re-executar um único índice de um Indexed Job sem recriar o Job completo. A abordagem Staff-level é: ao detectar falha permanente de um índice, criar um Job separado com `completions: 1` e `JOB_COMPLETION_INDEX` fixo via env var — o código do worker deve aceitar o índice tanto do env padrão quanto de uma variável override.
