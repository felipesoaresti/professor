---
tags:
  - exercicios
  - observabilidade
  - prometheus
  - grafana
  - alertmanager
  - promql
tipo: exercicios
area: observabilidade
conteudo: "[[06-Observabilidade/kube-prometheus-stack]]"
trilha: "[[00-Trilha/observabilidade]]"
---

# Exercícios — kube-prometheus-stack, Grafana e AlertManager

> Conteúdo: [[06-Observabilidade/kube-prometheus-stack]] | Trilha: [[00-Trilha/observabilidade]]

**Infraestrutura:** k8s-master (192.168.3.30), worker1 (.31), worker2 (.32)
**Namespace do stack:** `monitoring` (criar durante os exercícios)
**Namespace de app:** `estudo`
**Ingress:** nginx em 192.168.3.100 → `*.staypuff.info`
**Storage:** `nfs-homelab` para persistência do Prometheus e Grafana
**Atenção:** Nunca tocar em `controle-gastos`, `promobot`, `databases/postgres`.

---

## Exercício 1: Instalar o kube-prometheus-stack via Helm

**Contexto:** O time de plataforma decidiu adotar o kube-prometheus-stack como stack padrão de observabilidade. Você é o responsável pela instalação inicial no homelab.

**Missão:** Instale o stack completo via Helm com persistência no NFS e Ingress para Grafana.

**Requisitos:**
- [ ] Adicionar o repositório Helm: `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update`
- [ ] Criar um arquivo `prometheus-values.yaml` com:
  - Grafana com Ingress em `grafana.staypuff.info` (TLS via cert-manager, ClusterIssuer production)
  - Senha do Grafana definida (não usar a default)
  - Persistência do Grafana: StorageClass `nfs-homelab`, 5Gi
  - Prometheus: retenção de 30 dias, StorageClass `nfs-homelab`, 20Gi
  - AlertManager: StorageClass `nfs-homelab`, 2Gi
  - `serviceMonitorSelectorNilUsesHelmValues: false` (monitorar todos os ServiceMonitors do cluster)
- [ ] Instalar: `helm install kube-prometheus-stack ... --namespace monitoring --create-namespace -f prometheus-values.yaml`
- [ ] Aguardar todos os Pods ficarem Running: `kubectl get pods -n monitoring -w`
- [ ] Verificar os CRDs criados: `kubectl get crd | grep monitoring`
- [ ] Verificar os PVCs criados: `kubectl get pvc -n monitoring`
- [ ] Acessar o Grafana via Ingress e confirmar login com a senha configurada

**Verificação:**
```bash
kubectl get pods -n monitoring
# prometheus-kube-prometheus-stack-prometheus-0: Running
# alertmanager-kube-prometheus-stack-alertmanager-0: Running
# kube-prometheus-stack-grafana-xxx: Running
# kube-prometheus-stack-kube-state-metrics-xxx: Running
# kube-prometheus-stack-prometheus-node-exporter-xxx (um por node): Running

kubectl get pvc -n monitoring
# prometheus-kube-prometheus-stack-prometheus-db-0: Bound, 20Gi
# grafana: Bound, 5Gi

kubectl get crd | grep monitoring | wc -l
# Pelo menos 5 CRDs
```

---

## Exercício 2: Explorar o Prometheus UI e os targets pré-configurados

**Contexto:** O stack instalou automaticamente ServiceMonitors para todos os componentes do Kubernetes (kube-apiserver, etcd, kubelet, kube-state-metrics, node-exporter). Você precisa entender o que já está sendo monitorado antes de adicionar suas próprias aplicações.

**Missão:** Explore a UI do Prometheus e mapeie todos os targets ativos.

**Requisitos:**
- [ ] Acessar o Prometheus via port-forward: `kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090`
- [ ] Em `Status → Targets`: contar quantos targets estão UP e identificar os grupos (jobs)
- [ ] Identificar as métricas expostas pelo `node-exporter` de cada worker:
  - Qual o job name do node-exporter?
  - Quais labels cada sample tem? (instance, node, job)
- [ ] Em `Status → Service Discovery`: ver quais ServiceMonitors foram descobertos
- [ ] Em `Status → Rules`: quais grupos de regras de alerta estão pré-configurados?
- [ ] Executar as seguintes queries no campo de expressão:
  - `up` → ver todos os targets e seus estados
  - `kube_node_info` → ver informações dos nodes
  - `kube_pod_info{namespace="databases"}` → ver os Pods do namespace databases
- [ ] Em `Alerts`: há algum alerta em estado `Firing` agora?

**Verificação:**
```promql
# No Prometheus UI, executar:
up{job="kube-state-metrics"}
# Deve retornar 1 (UP)

count(up == 1)
# Número total de targets UP

kube_node_status_condition{condition="Ready", status="true"}
# Deve retornar 1 para cada node (3 no total)
```

---

## Exercício 3: PromQL — queries sobre CPU, memória e Pods

**Contexto:** O time de operações quer um relatório de uso de recursos do cluster. Você precisa construir as queries PromQL para responder perguntas específicas sobre o estado do cluster.

**Missão:** Escreva e execute 8 queries PromQL respondendo perguntas sobre o cluster.

**Requisitos:**
- [ ] **Q1:** Qual é o uso atual de CPU (em cores) de cada node? (`node_cpu_seconds_total`)
- [ ] **Q2:** Qual é o uso de memória (em GB) de cada node? (`node_memory_MemAvailable_bytes`, `node_memory_MemTotal_bytes`)
- [ ] **Q3:** Quantos Pods estão rodando em cada namespace? (`kube_pod_info`)
- [ ] **Q4:** Quais Pods têm `requests.memory` definido maior que 128Mi? (`kube_pod_container_resource_requests`)
- [ ] **Q5:** Qual é a porcentagem de CPU utilizada por cada Pod no namespace `databases`? (`container_cpu_usage_seconds_total`)
- [ ] **Q6:** Quantos restarts os containers tiveram hoje? (`kube_pod_container_status_restarts_total`)
- [ ] **Q7:** Qual a quantidade de disco usada em cada node? (`node_filesystem_size_bytes`, `node_filesystem_free_bytes`)
- [ ] **Q8:** Use `predict_linear` para estimar quando o disco do master ficará cheio baseado na tendência das últimas 2 horas
- [ ] Para cada query: registrar o resultado e entender o que os labels retornados significam

**Verificação:**
```promql
# Q3: Pods por namespace
count by (namespace) (kube_pod_info)
# databases: 1, kube-system: N, monitoring: N, ...

# Q6: Containers com mais restarts
topk(5, kube_pod_container_status_restarts_total)
# Mostra os 5 containers com mais restarts

# Q8: Estimativa de disco cheio
predict_linear(node_filesystem_free_bytes{mountpoint="/", instance="k8s-master:9100"}[2h], 7*24*3600)
# Valor esperado em 7 dias — se negativo, o disco encheria antes
```

---

## Exercício 4: ServiceMonitor para uma aplicação customizada

**Contexto:** O time desenvolveu uma aplicação que expõe métricas no formato Prometheus em `:8080/metrics`. Você precisa configurar o scraping automático via ServiceMonitor.

**Missão:** Deploy de uma aplicação com métricas e configuração do ServiceMonitor.

**Requisitos:**
- [ ] Criar namespace `estudo`
- [ ] Fazer deploy de uma aplicação que exponha métricas — usar `prom/prometheus` como gerador de métricas próprias:
  ```yaml
  # Alternativa simples: nginx com stub_status + exporter
  # Ou: usar a imagem prom/node-exporter como "aplicação de teste"
  kubectl create deployment app-metrics -n estudo \
    --image=prom/node-exporter:latest \
    --port=9100
  kubectl expose deployment app-metrics -n estudo \
    --port=9100 --name=app-metrics-svc \
    --target-port=9100
  kubectl label svc app-metrics-svc -n estudo app=app-metrics
  ```
- [ ] Adicionar nome à porta no Service (ServiceMonitor exige porta nomeada):
  ```bash
  kubectl patch svc app-metrics-svc -n estudo \
    --type='json' \
    -p='[{"op":"add","path":"/spec/ports/0/name","value":"metrics"}]'
  ```
- [ ] Criar ServiceMonitor com label `release: kube-prometheus-stack`:
  - `selector.matchLabels: {app: app-metrics}`
  - `endpoints[].port: metrics`
  - `namespaceSelector.matchNames: [estudo]`
- [ ] Aguardar 30–60s e verificar no Prometheus UI em `Status → Targets` se o novo target aparece como UP
- [ ] Executar uma query sobre as métricas do novo target: `up{job="estudo/app-metrics"}`
- [ ] **Desafio:** E se o label `release` estiver errado? Mudar para `release: errado` e observar o que acontece

**Verificação:**
```bash
kubectl get servicemonitor -n estudo
# app-metrics: criado

# No Prometheus UI, após ~60s:
# Status → Targets → deve aparecer "estudo/app-metrics" como UP

# Query:
up{namespace="estudo"}
# 1 = UP
```

---

## Exercício 5: PrometheusRule — criar alertas e verificar no Prometheus

**Contexto:** O time quer ser notificado quando qualquer Pod no namespace `estudo` ficar não-ready por mais de 2 minutos, e quando o uso de memória ultrapassar 80% do limit.

**Missão:** Crie PrometheusRules, force uma condição de alerta e observe o ciclo completo.

**Requisitos:**
- [ ] Criar `PrometheusRule` no namespace `estudo` com label `release: kube-prometheus-stack` contendo:
  - **Alerta 1:** `PodNaoReady` — Pod com `kube_pod_status_ready{condition="false"} == 1` por mais de 2 minutos, severity: `critical`
  - **Alerta 2:** `DeploymentSemReplicas` — `kube_deployment_spec_replicas == 0` por mais de 1 minuto no namespace `estudo`, severity: `warning`
  - **Recording rule:** `namespace:kube_pod_count:sum` = `count by (namespace) (kube_pod_info)`
- [ ] Verificar que as regras foram carregadas: `Status → Rules` no Prometheus UI
- [ ] **Forçar o alerta:** criar um Deployment com 0 réplicas:
  ```bash
  kubectl create deployment teste-alerta -n estudo --image=nginx:1.27 --replicas=0
  ```
- [ ] Observar o alerta passar de `Inactive → Pending → Firing` no Prometheus UI (`Alerts` tab)
- [ ] Quanto tempo levou para ir de Pending para Firing? (deve ser ~1 minuto conforme `for:`)
- [ ] Verificar que a recording rule gerou uma métrica consultável:
  ```promql
  namespace:kube_pod_count:sum
  ```
- [ ] Corrigir: `kubectl scale deployment teste-alerta -n estudo --replicas=1`
- [ ] Observar o alerta resolver (voltar a `Inactive`)

**Verificação:**
```bash
kubectl get prometheusrule -n estudo
# minha-app-alerts: criado

# No Prometheus UI → Alerts:
# DeploymentSemReplicas: Pending → Firing (após 1min)

# Recording rule:
namespace:kube_pod_count:sum{namespace="estudo"}
# Retorna o número de Pods no namespace estudo
```

---

## Exercício 6: Grafana — explorar dashboards built-in e criar um painel

**Contexto:** O time quer um dashboard centralizado para monitorar o cluster. O kube-prometheus-stack já inclui dashboards prontos — você vai explorá-los e criar um painel customizado para o namespace `estudo`.

**Missão:** Explore dashboards existentes, importe um da comunidade, e crie um painel manualmente.

**Requisitos:**
- [ ] Acessar o Grafana em `grafana.staypuff.info` (ou via port-forward)
- [ ] Navegar em `Dashboards → Browse` e listar os dashboards pré-instalados pelo kube-prometheus-stack
- [ ] Abrir o dashboard `Kubernetes / Compute Resources / Namespace (Pods)` e selecionar o namespace `estudo`
  - Quais métricas ele mostra?
  - Qual a query PromQL do painel de CPU?
- [ ] Importar o dashboard `Node Exporter Full` (ID: **1860**) via `Dashboards → Import`
- [ ] Criar um dashboard novo `Estudo — Visão Geral` com 3 painéis:
  - **Painel 1 (Stat):** Número de Pods Running no namespace `estudo`
    - Query: `count(kube_pod_status_phase{namespace="estudo", phase="Running"})`
  - **Painel 2 (Time Series):** CPU usage dos Pods em `estudo` (últimos 30min)
    - Query: `sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="estudo", container!=""}[5m]))`
  - **Painel 3 (Table):** Lista de Pods e seus estados
    - Query: `kube_pod_status_phase{namespace="estudo"}`
- [ ] Salvar o dashboard e exportar como JSON (Share → Export)

**Verificação:**
```
Grafana → Dashboards → Browse:
- Kubernetes dashboards (CPU, Memory, Network, etc.) pré-instalados

Dashboard "Estudo — Visão Geral":
- Painel 1 mostra número atual de Pods Running
- Painel 2 mostra linha de CPU por Pod
- Painel 3 mostra tabela com fases dos Pods
```

---

## Exercício 7: AlertManager — configurar receiver e testar notificação

**Contexto:** Os alertas estão sendo gerados no Prometheus, mas o AlertManager está configurado com receiver `null` (sem notificação real). O time quer receber alertas no Telegram quando qualquer alerta `critical` disparar.

**Missão:** Configure um receiver de Telegram (ou Slack/webhook) e valide o envio de uma notificação real.

**Requisitos:**
- [ ] Acessar a UI do AlertManager: `kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093`
- [ ] Verificar a config atual: ver se há algum receiver configurado além de `null`
- [ ] Criar um bot do Telegram (via @BotFather) e obter `bot_token` e `chat_id`
  - Alternativa: usar um webhook público de teste como `https://webhook.site`
- [ ] Atualizar os values do Helm para configurar o receiver:
  ```yaml
  alertmanager:
    config:
      global:
        resolve_timeout: 5m
      route:
        receiver: telegram
        group_by: ['alertname', 'namespace']
        group_wait: 30s
        repeat_interval: 4h
      receivers:
      - name: telegram
        telegram_configs:
        - bot_token: '<TOKEN>'
          chat_id: <CHAT_ID>
          message: "🚨 {{ .GroupLabels.alertname }} | {{ range .Alerts }}{{ .Annotations.summary }}{{ end }}"
  ```
- [ ] Aplicar: `helm upgrade kube-prometheus-stack ... -f prometheus-values.yaml -n monitoring`
- [ ] **Testar:** Forçar o alerta do Exercício 5 (`DeploymentSemReplicas`)
- [ ] Verificar no AlertManager UI (`Status → Alerts`) se o alerta chegou
- [ ] Confirmar que a notificação foi recebida no Telegram/webhook
- [ ] Criar um silêncio para o alerta via UI do AlertManager (Silences → New Silence)
- [ ] Verificar que o alerta silenciado não gera nova notificação

**Verificação:**
```bash
# AlertManager UI → Status → Alerts:
# DeploymentSemReplicas: state=active, receiver=telegram

# Telegram ou webhook:
# Mensagem recebida com o texto configurado

# Após criar Silence:
# AlertManager UI → Silences → alerta aparece como silenciado
# Sem nova notificação mesmo com o alerta ativo
```

---

## Exercício 8: Dashboard como código — provisionamento via ConfigMap (Staff-level)

**Contexto:** O time de plataforma quer que dashboards de observabilidade das aplicações sejam versionados em git e provisionados automaticamente no Grafana — sem intervenção manual na UI. Qualquer novo cluster deve ter os mesmos dashboards automaticamente.

**Missão:** Implemente um dashboard completo como ConfigMap provisionado automaticamente pelo Grafana.

**Requisitos:**
- [ ] Verificar que o sidecar do Grafana está habilitado para ler ConfigMaps com label `grafana_dashboard: "1"`:
  ```bash
  kubectl get deployment kube-prometheus-stack-grafana -n monitoring -o yaml | grep -A5 sidecar
  ```
- [ ] Criar um ConfigMap no namespace `monitoring` com label `grafana_dashboard: "1"` contendo um dashboard JSON completo para o namespace `estudo`:
  - Título: `Estudo — Observabilidade`
  - UID: `estudo-obs`
  - 4 painéis: CPU (time series), Memória (time series), Pods Running (stat), Restarts (bar)
  - Variável de template para selecionar namespace
- [ ] Aplicar o ConfigMap e aguardar o sidecar do Grafana detectar (verificar os logs do sidecar):
  ```bash
  kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard -f
  ```
- [ ] Confirmar que o dashboard apareceu no Grafana sem restart manual
- [ ] **Desafio:** exportar o JSON de um dashboard existente no Grafana (Share → Export → Save to file) e provisioná-lo via ConfigMap
- [ ] Deletar o ConfigMap e confirmar que o dashboard desaparece do Grafana
- [ ] Limpeza: os recursos de `estudo` podem ser deletados, mas manter o stack `monitoring` para uso futuro

**Verificação:**
```bash
kubectl get configmap -n monitoring | grep grafana-dashboard
# minha-app-dashboard: presente

kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard | grep "estudo-obs"
# Loading dashboard from ConfigMap ...

# Grafana → Dashboards → Browse:
# "Estudo — Observabilidade" aparece automaticamente
```

---

> [!tip] Ordem recomendada
> Exercício 1 é a base — sem o stack instalado, nada funciona.
> Exercícios 2 e 3 são exploratórios — fundamentais para entender PromQL antes de criar alertas.
> Exercício 4 antes do 5 — você precisa de uma aplicação sendo monitorada para ver os efeitos das regras.
> Exercícios 6 e 7 são independentes entre si — podem ser feitos em qualquer ordem após o 3.
> Exercício 8 é Staff-level — só faz sentido depois de dominar o Grafana manualmente.

> [!warning] Storage NFS para Prometheus
> O Prometheus com 30 dias de retenção pode acumular vários GBs de dados. Verifique o espaço disponível no NFS antes de instalar: `df -h 192.168.3.11:/mnt/nfs-data/k8s` (ou conectar via SSH ao servidor NFS).

> [!warning] Rate limit do AlertManager
> O AlertManager tem `repeat_interval` configurado. Se você forçar alertas repetidamente para testar, ajuste `repeat_interval: 1m` durante o desenvolvimento e restaure para `4h` em produção.

> [!info] Grafana default credentials
> O kube-prometheus-stack instala com usuário `admin` e senha `prom-operator` por padrão. Sempre trocar a senha via `grafana.adminPassword` no values.yaml antes de expor via Ingress.
