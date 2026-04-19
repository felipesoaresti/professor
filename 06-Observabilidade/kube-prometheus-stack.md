---
tags:
  - observabilidade
  - prometheus
  - grafana
  - alertmanager
  - kube-prometheus
  - promql
area: observabilidade
tipo: conteudo
prerequisites:
  - "[[05-Kubernetes/kubernetes-teoria-inicial]]"
  - "[[05-Kubernetes/services]]"
  - "[[05-Kubernetes/labels-annotations]]"
next:
  - "[[11-Exercicios/kube-prometheus-stack]]"
trilha: "[[00-Trilha/observabilidade]]"
---

# kube-prometheus-stack — Prometheus, Grafana e AlertManager

> Pré-requisitos: [[05-Kubernetes/kubernetes-teoria-inicial]] | [[05-Kubernetes/services]] | Exercícios: [[11-Exercicios/kube-prometheus-stack]] | Trilha: [[00-Trilha/observabilidade]]

---

## O que é e por que existe

Kubernetes sem observabilidade é uma caixa preta. Você sabe que os Pods estão `Running`, mas não sabe se a aplicação está respondendo, se a memória está crescendo, ou se a latência do banco subiu 10x às 3h da manhã.

O **kube-prometheus-stack** é um Helm chart que empacota o ecossistema completo de observabilidade para Kubernetes:

| Componente | Função |
|---|---|
| **Prometheus Operator** | Gerencia o ciclo de vida do Prometheus via CRDs |
| **Prometheus** | Coleta e armazena métricas em série temporal (TSDB) |
| **AlertManager** | Roteamento, deduplicação e envio de alertas |
| **Grafana** | Dashboards e visualização das métricas |
| **kube-state-metrics** | Expõe estado dos objetos K8s como métricas |
| **node-exporter** | Expõe métricas de hardware de cada node (DaemonSet) |
| **Prometheus Adapter** | Expõe métricas custom para HPA |

> [!info] Por que "kube-prometheus-stack" e não só "Prometheus"?
> Prometheus puro não conhece Kubernetes. O Operator adiciona CRDs (`ServiceMonitor`, `PodMonitor`, `PrometheusRule`) que permitem configurar scraping e alertas de forma declarativa — sem reescrever o `prometheus.yml` a cada mudança.

---

## Como funciona internamente

### Prometheus Operator — os CRDs que mudam tudo

O Operator adiciona 5 CRDs principais ao cluster:

```
ServiceMonitor   → define como fazer scrape de um Service (via Endpoints)
PodMonitor       → define como fazer scrape direto em Pods
PrometheusRule   → define regras de alerta e recording rules
Alertmanager     → configura uma instância do AlertManager
Prometheus       → configura uma instância do Prometheus
```

O fluxo sem Operator (estático):
```
prometheus.yml atualizado manualmente → restart do Prometheus
```

O fluxo com Operator (dinâmico):
```
ServiceMonitor criado → Operator detecta → regenera config → Prometheus reload automático
```

### Como o Prometheus coleta métricas

Prometheus usa o modelo **pull**: ele consulta os targets periodicamente (padrão: 15s), não espera que as aplicações enviem dados:

```
Prometheus
  │  a cada 15s (scrapeInterval)
  ├── GET http://node-exporter:9100/metrics
  ├── GET http://kube-state-metrics:8080/metrics
  ├── GET http://minha-app:8080/metrics       ← via ServiceMonitor
  └── armazena no TSDB local (chunks em disco)
```

Cada série temporal é identificada por um conjunto de **labels**:
```
http_requests_total{method="GET", status="200", pod="webapp-abc", namespace="estudo"}
```

### TSDB — o banco de séries temporais

O Prometheus armazena dados em blocos de 2 horas (em memória) e depois compacta para disco em blocos de até 12 horas. Por padrão, retém dados por **15 dias**.

```
/prometheus/data/
├── 01ABCDEF.../   ← bloco de 2h compactado
│   ├── chunks/
│   ├── index
│   └── meta.json
├── wal/           ← Write-Ahead Log (dados das últimas 2h em memória)
└── ...
```

Para retenção longa (meses/anos) → Thanos ou VictoriaMetrics.

### ServiceMonitor — como o Prometheus descobre sua aplicação

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: minha-app-monitor
  namespace: estudo
  labels:
    release: kube-prometheus-stack   # ← label que o Prometheus Operator procura
spec:
  selector:
    matchLabels:
      app: minha-app                 # encontra Services com este label
  endpoints:
  - port: http-metrics               # nome da porta no Service
    path: /metrics
    interval: 15s
  namespaceSelector:
    matchNames:
    - estudo
```

> [!warning] O label `release: kube-prometheus-stack` é crítico
> O Prometheus instalado pelo Helm só monitora ServiceMonitors que tenham o label `release: kube-prometheus-stack` (ou o valor que você configurou em `prometheus.prometheusSpec.serviceMonitorSelector`). Sem esse label, o ServiceMonitor é silenciosamente ignorado.

### Pipeline do AlertManager

```
PrometheusRule dispara alerta
       │
       ▼
Prometheus avalia a regra a cada `evaluationInterval` (padrão: 15s)
       │  se a condição for verdadeira por `for: 5m` → estado Firing
       ▼
AlertManager recebe o alerta
       │
       ├── Grouping    → agrupa alertas similares (evita flood)
       ├── Inhibition  → suprime alertas secundários quando o primário já disparou
       ├── Silence     → suprime alertas por período configurado
       │
       ▼
Receiver (Slack, Telegram, PagerDuty, email, webhook)
```

### Grafana — como os dashboards funcionam

O Grafana se conecta ao Prometheus via **data source** e executa queries PromQL para renderizar os painéis. Cada painel tem:
- Uma ou mais queries PromQL
- Tipo de visualização (time series, gauge, stat, table, heatmap)
- Thresholds e alertas (deprecated em favor do AlertManager)

Dashboards podem ser:
- **Manuais**: criados na UI e exportados como JSON
- **Provisionados**: JSON em ConfigMap lido automaticamente pelo Grafana no boot
- **Importados**: via ID do Grafana.com (ex: Dashboard 1860 = Node Exporter Full)

---

## Na prática — instalação e configuração

### Instalação via Helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# values mínimos para o homelab
cat > prometheus-values.yaml << 'EOF'
grafana:
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-production
    hosts:
    - grafana.staypuff.info
    tls:
    - secretName: grafana-tls
      hosts:
      - grafana.staypuff.info
  adminPassword: "SenhaForte123!"
  persistence:
    enabled: true
    storageClassName: nfs-homelab
    size: 5Gi

prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-homelab
          resources:
            requests:
              storage: 20Gi
    serviceMonitorSelectorNilUsesHelmValues: false  # monitorar TODOS os ServiceMonitors
    podMonitorSelectorNilUsesHelmValues: false

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-homelab
          resources:
            requests:
              storage: 2Gi
EOF

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f prometheus-values.yaml
```

### Verificar o que foi instalado

```bash
kubectl get pods -n monitoring
# prometheus-kube-prometheus-stack-prometheus-0   (StatefulSet)
# alertmanager-kube-prometheus-stack-alertmanager-0
# kube-prometheus-stack-grafana-xxx
# kube-prometheus-stack-kube-state-metrics-xxx
# kube-prometheus-stack-prometheus-node-exporter-xxx (DaemonSet — um por node)

kubectl get crd | grep monitoring
# alertmanagers.monitoring.coreos.com
# podmonitors.monitoring.coreos.com
# prometheusrules.monitoring.coreos.com
# servicemonitors.monitoring.coreos.com

kubectl get servicemonitor -n monitoring
# Várias ServiceMonitors pré-configurados para componentes do K8s
```

### Criar ServiceMonitor para sua aplicação

A aplicação precisa expor métricas no formato Prometheus (texto puro) em `/metrics`:

```yaml
# Service da aplicação com porta nomeada
apiVersion: v1
kind: Service
metadata:
  name: minha-app-svc
  namespace: estudo
  labels:
    app: minha-app
spec:
  selector:
    app: minha-app
  ports:
  - name: http         # porta da aplicação
    port: 80
    targetPort: 80
  - name: http-metrics # porta de métricas
    port: 8080
    targetPort: 8080
---
# ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: minha-app
  namespace: estudo
  labels:
    release: kube-prometheus-stack  # OBRIGATÓRIO
spec:
  selector:
    matchLabels:
      app: minha-app
  endpoints:
  - port: http-metrics
    path: /metrics
    interval: 30s
  namespaceSelector:
    matchNames:
    - estudo
```

### Verificar no Prometheus UI

```bash
# Port-forward para acessar a UI
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# Acessar http://localhost:9090
# Status → Targets: verificar se sua aplicação aparece como UP
# Status → Service Discovery: ver quais ServiceMonitors foram carregados
```

### PromQL — queries fundamentais

```promql
# Taxa de requisições HTTP por segundo (últimos 5min)
rate(http_requests_total[5m])

# Taxa por status code e namespace
sum by (status, namespace) (rate(http_requests_total[5m]))

# Percentil 99 de latência (requer histograma)
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# Uso de CPU por Pod (em cores)
sum by (pod, namespace) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))

# Uso de memória por Pod (em bytes)
sum by (pod, namespace) (container_memory_working_set_bytes{container!=""})

# Pods não-ready por namespace
kube_pod_status_ready{condition="false"} == 1

# Nodes com menos de 10% de CPU disponível
1 - (sum by (node) (rate(node_cpu_seconds_total{mode="idle"}[5m])) / sum by (node) (rate(node_cpu_seconds_total[5m])))

# Uso de disco do PVC
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes

# Disponibilidade de um Deployment (réplicas prontas / desejadas)
kube_deployment_status_replicas_ready / kube_deployment_spec_replicas
```

### PrometheusRule — definir alertas

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: minha-app-alerts
  namespace: estudo
  labels:
    release: kube-prometheus-stack  # OBRIGATÓRIO
spec:
  groups:
  - name: minha-app.rules
    interval: 30s
    rules:
    # Alerta: Pod down por mais de 2 minutos
    - alert: PodNaoDisponivel
      expr: kube_pod_status_ready{namespace="estudo", condition="false"} == 1
      for: 2m
      labels:
        severity: critical
        namespace: "{{ $labels.namespace }}"
      annotations:
        summary: "Pod {{ $labels.pod }} não está pronto"
        description: "O Pod {{ $labels.pod }} no namespace {{ $labels.namespace }} está não-ready por mais de 2 minutos."
        runbook: "https://wiki.staypuff.info/runbooks/pod-not-ready"

    # Alerta: Uso de CPU alto
    - alert: AltoCPU
      expr: |
        sum by (pod, namespace) (rate(container_cpu_usage_seconds_total{container!="", namespace="estudo"}[5m]))
        / sum by (pod, namespace) (kube_pod_container_resource_limits{resource="cpu", namespace="estudo"})
        > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} com CPU acima de 80%"
        description: "CPU usage {{ $value | humanizePercentage }} por mais de 5 minutos."

    # Recording rule — pre-computa query cara
    - record: job:http_requests_total:rate5m
      expr: sum by (job, namespace) (rate(http_requests_total[5m]))
```

### AlertManager — configurar receivers

```yaml
# ConfigMap ou Secret do AlertManager (via Helm values)
alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s        # aguardar antes de enviar (agrupa alertas simultâneos)
      group_interval: 5m     # intervalo entre notificações do mesmo grupo
      repeat_interval: 12h   # reenviar se ainda estiver disparado
      receiver: 'telegram'
      routes:
      - matchers:
        - severity = critical
        receiver: 'telegram'
        repeat_interval: 1h
      - matchers:
        - severity = warning
        receiver: 'slack'
    receivers:
    - name: 'telegram'
      telegram_configs:
      - bot_token: '<BOT_TOKEN>'
        chat_id: <CHAT_ID>
        message: |
          🚨 *{{ .GroupLabels.alertname }}*
          Namespace: `{{ .GroupLabels.namespace }}`
          {{ range .Alerts }}
          • {{ .Annotations.summary }}
          {{ end }}
    - name: 'slack'
      slack_configs:
      - api_url: '<SLACK_WEBHOOK_URL>'
        channel: '#alertas-k8s'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    inhibit_rules:
    - source_matchers:
      - severity = critical
      target_matchers:
      - severity = warning
      equal: ['alertname', 'namespace']   # suprime warning se critical do mesmo alerta já disparou
```

### Grafana — importar e criar dashboards

```bash
# Acessar via Ingress ou port-forward
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80

# Dashboards populares (importar pelo ID em Grafana.com):
# 1860  → Node Exporter Full (métricas de hardware dos nodes)
# 15760 → Kubernetes / Views / Global
# 12740 → Kubernetes Pod Logs
# 13332 → Kube State Metrics
# 14981 → Kubernetes Deployments
```

### Dashboard como código — provisionamento via ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: minha-app-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"   # label que o Grafana sidecar detecta automaticamente
data:
  minha-app.json: |
    {
      "title": "Minha App — Visão Geral",
      "uid": "minha-app-overview",
      "panels": [
        {
          "title": "Taxa de Requisições",
          "type": "timeseries",
          "targets": [
            {
              "expr": "sum(rate(http_requests_total{namespace=\"estudo\"}[5m]))",
              "legendFormat": "req/s"
            }
          ]
        }
      ],
      "refresh": "30s",
      "time": {"from": "now-1h", "to": "now"}
    }
```

O sidecar do Grafana (`grafana-sc-dashboard`) monitora ConfigMaps com o label `grafana_dashboard: "1"` e injeta o JSON automaticamente — sem restart do Grafana.

---

## Casos de uso e boas práticas

**Separe namespaces de monitoring e aplicações** — ServiceMonitors em diferentes namespaces com `namespaceSelector` correto.

**Use recording rules para queries pesadas** — queries com `histogram_quantile` sobre longos períodos são caras. Pré-compute com recording rules avaliadas a cada 1min.

**Nomeie alertas de forma acionável** — o nome do alerta deve descrever o problema, não a métrica: `PodNaoDisponivel` em vez de `kube_pod_ready_false`.

**`for:` adequado ao SLO** — `for: 1m` gera muitos false positives em deploys. Para alertas críticos, use `for: 5m`. Para warning, `for: 15m`.

**Runbook em toda annotation de alerta** — adicionar `annotations.runbook` com URL do procedimento de resolução. Quando o alerta dispara às 3h, o plantão precisa saber o que fazer.

**Retenção e custo de storage** — 15 dias é o padrão. Para homelab, 30 dias é razoável. Calcule: ~1-2 bytes/sample × 15 samples/min × número de séries.

**Evite cardinalidade alta** — cada combinação única de labels = uma série temporal. Nunca use IDs de usuário ou request IDs como labels do Prometheus — explode o TSDB.

---

## Troubleshooting — cenários reais de produção

### Target DOWN no Prometheus UI

```
Status → Targets → encontrar o target com estado "DOWN"
```

Causas comuns:
1. **ServiceMonitor com label errado** — falta `release: kube-prometheus-stack`
2. **Porta não nomeada** — a porta no Service deve ter o mesmo nome que o ServiceMonitor referencia em `endpoints[].port`
3. **Path errado** — aplicação não expõe `/metrics` ou usa outro path
4. **NetworkPolicy** bloqueando o scrape — Prometheus precisa de ingress do namespace `monitoring` para a porta de métricas

```bash
# Verificar se o ServiceMonitor foi carregado
kubectl get servicemonitor -n estudo
kubectl describe servicemonitor minha-app -n estudo

# Ver a config gerada pelo Operator
kubectl exec -n monitoring <prometheus-pod> -- \
  cat /etc/prometheus/config_out/prometheus.env.yaml | grep -A10 minha-app

# Testar o endpoint de métricas manualmente
kubectl exec -n monitoring <prometheus-pod> -- \
  wget -qO- http://minha-app-svc.estudo.svc.cluster.local:8080/metrics | head -20
```

### AlertManager não dispara

```bash
# Verificar se o alerta está FIRING no Prometheus
# Alerts → ver se está em Pending ou Firing

# Verificar a config do AlertManager
kubectl get secret -n monitoring alertmanager-kube-prometheus-stack-alertmanager -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d

# Logs do AlertManager
kubectl logs -n monitoring -l app=alertmanager --tail=50

# Simular envio de alerta (via curl à API do AlertManager)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093
curl -X POST http://localhost:9093/api/v1/alerts -d '[{"labels":{"alertname":"Teste","severity":"critical"}}]'
```

### Grafana sem dados ("No data")

1. **Data source mal configurado** — acessar `Configuration → Data Sources → Test`
2. **Query PromQL com erro** — verificar no Explore com `Shift+Enter`
3. **Namespace ou label errado na query** — usar `label_values(kube_pod_info, namespace)` para confirmar os valores reais
4. **Intervalo de tempo muito curto** — aumentar o range no dashboard

### Prometheus com TSDB cheio

```bash
kubectl exec -n monitoring <prometheus-pod> -- df -h /prometheus
# Se /prometheus está cheio → TSDB parou de aceitar novos dados

# Ver o tamanho atual do TSDB
kubectl exec -n monitoring <prometheus-pod> -- \
  promtool tsdb analyze /prometheus

# Opções: aumentar o PVC, reduzir retenção, ou adicionar storage remoto
```

---

## Nível avançado — edge cases e Staff-level

### PromQL avançado

```promql
# Alertas que ficam pending mas não disparam (útil para debug)
ALERTS{alertstate="pending"}

# Top 5 Pods com maior uso de memória
topk(5, sum by (pod, namespace) (container_memory_working_set_bytes{container!=""}))

# Predição linear: em quantas horas o disco estará cheio?
predict_linear(node_filesystem_free_bytes{mountpoint="/"}[1h], 4*3600) < 0

# Absent: alerta quando uma métrica some (Pod crashou e /metrics sumiu)
absent(up{job="minha-app"})

# Comparação entre períodos (hoje vs ontem)
http_requests_total - http_requests_total offset 24h
```

### Recording rules para SLO

```yaml
- name: slo.rules
  rules:
  # Disponibilidade do webapp nos últimos 30 dias
  - record: slo:webapp_availability:ratio_rate30d
    expr: |
      sum(rate(http_requests_total{namespace="estudo", status!~"5.."}[30d]))
      / sum(rate(http_requests_total{namespace="estudo"}[30d]))

  # Error budget restante (SLO de 99.9%)
  - record: slo:webapp_error_budget_remaining:ratio
    expr: |
      1 - ((1 - slo:webapp_availability:ratio_rate30d) / (1 - 0.999))
```

### Thanos — retenção longa (homelab opcional)

```bash
# Thanos Sidecar injeta dados do Prometheus para object storage (S3/MinIO)
# Thanos Query agrega múltiplos Prometheuses
# Thanos Compactor compacta dados antigos

# Para homelab: VictoriaMetrics é mais simples e eficiente
helm install victoria-metrics vm/victoria-metrics-single \
  --set server.retentionPeriod=12 \  # 12 meses
  --set server.persistentVolume.storageClass=nfs-homelab
```

### Alertas de SLO com Pyrra ou Sloth

```bash
# Sloth gera PrometheusRules de SLO automaticamente
# Dado um SLO de 99.9% de disponibilidade:
apiVersion: sloth.slok.dev/v1
kind: PrometheusServiceLevel
metadata:
  name: webapp-slo
spec:
  service: webapp
  slos:
  - name: availability
    objective: 99.9
    sli:
      events:
        errorQuery: sum(rate(http_requests_total{status=~"5..", namespace="estudo"}[{{.window}}]))
        totalQuery: sum(rate(http_requests_total{namespace="estudo"}[{{.window}}]))
    alerting:
      pageAlert:
        labels:
          severity: critical
      ticketAlert:
        labels:
          severity: warning
```

### Federação — um Prometheus central para múltiplos clusters

```yaml
# prometheus-central.yaml
scrape_configs:
- job_name: 'federate'
  scrape_interval: 30s
  honor_labels: true
  metrics_path: /federate
  params:
    match[]:
    - '{job="kube-state-metrics"}'
    - '{__name__=~"job:.*"}'  # apenas recording rules
  static_configs:
  - targets:
    - prometheus-cluster1:9090
    - prometheus-cluster2:9090
```

---

## Referências

- [Prometheus Docs](https://prometheus.io/docs/)
- [Prometheus Operator Docs](https://prometheus-operator.dev/docs/)
- [kube-prometheus-stack Helm Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Grafana Docs — Dashboard Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- [AlertManager Docs](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [PromQL Cheat Sheet](https://promlabs.com/promql-cheat-sheet/)
- [Grafana Dashboards — grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards/)
- [Sloth — SLO Generator](https://github.com/slok/sloth)
- Relacionados: [[05-Kubernetes/labels-annotations]] | [[05-Kubernetes/services]] | [[05-Kubernetes/probes]]
