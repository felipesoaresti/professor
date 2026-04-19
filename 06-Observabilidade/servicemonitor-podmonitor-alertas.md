---
tags:
  - observabilidade
  - prometheus
  - servicemonitor
  - podmonitor
  - prometheusrule
  - alertas
  - prometheus-operator
area: observabilidade
tipo: conteudo
prerequisites:
  - "[[06-Observabilidade/kube-prometheus-stack]]"
  - "[[05-Kubernetes/labels-annotations]]"
next:
  - "[[11-Exercicios/servicemonitor-podmonitor-alertas]]"
trilha: "[[00-Trilha/observabilidade]]"
---

# ServiceMonitor, PodMonitor e PrometheusRule

> Pré-requisito: [[06-Observabilidade/kube-prometheus-stack]] | Exercícios: [[11-Exercicios/servicemonitor-podmonitor-alertas]] | Trilha: [[00-Trilha/observabilidade]]

---

## O que é e por que existe

Sem o Prometheus Operator, configurar scraping em Kubernetes exige editar manualmente o `prometheus.yml` e reiniciar o Prometheus a cada mudança. Com dezenas de aplicações e múltiplos times, isso não escala.

O Operator introduz três CRDs que tornam a observabilidade **declarativa e auto-gerenciada**:

| CRD | Propósito |
|---|---|
| **ServiceMonitor** | Define como fazer scrape de um Service (via Endpoints) |
| **PodMonitor** | Define como fazer scrape diretamente de Pods (sem Service) |
| **PrometheusRule** | Define regras de alerta e recording rules |

O fluxo:
```
Você cria um ServiceMonitor
        │
        ▼
Prometheus Operator detecta (watch no apiserver)
        │
        ▼
Operator regenera a config do Prometheus
        │
        ▼
Prometheus recarrega (sem restart) e começa a coletar
```

Cada time pode criar seus próprios ServiceMonitors e PrometheusRules no próprio namespace — sem acesso ao namespace `monitoring`.

---

## Como funciona internamente

### Como o Operator descobre os CRDs

O Prometheus Operator tem um campo `serviceMonitorSelector` que define quais ServiceMonitors ele monitora:

```yaml
# No objeto Prometheus (gerenciado pelo Helm)
spec:
  serviceMonitorSelector:
    matchLabels:
      release: kube-prometheus-stack   # padrão do Helm chart
```

Se `serviceMonitorSelector` for `{}` (vazio) ou `nil` com `serviceMonitorSelectorNilUsesHelmValues: false`, o Prometheus monitora **todos** os ServiceMonitors do cluster.

> [!tip] Configuração recomendada para o homelab
> Usar `serviceMonitorSelectorNilUsesHelmValues: false` no values.yaml do Helm elimina a necessidade do label `release:` em cada ServiceMonitor — qualquer ServiceMonitor criado em qualquer namespace é automaticamente descoberto.

### Como o Prometheus gera a config a partir do ServiceMonitor

O Operator traduz cada ServiceMonitor em um bloco `scrape_config` no `prometheus.yml`. Para verificar a config gerada:

```bash
kubectl exec -n monitoring <prometheus-pod> -- \
  cat /etc/prometheus/config_out/prometheus.env.yaml
```

Cada endpoint do ServiceMonitor gera um job com nome no formato:
```
<namespace>/<servicemonitor-name>/<endpoint-index>
```

### Como o scrape funciona — o caminho completo

```
ServiceMonitor
  selector → encontra Services com labels correspondentes
  endpoints[].port → resolve a porta no Service
        │
        ▼
Prometheus consulta o kube-apiserver para listar os Endpoints do Service
        │  (não é o DNS — é a API para obter os IPs dos Pods diretamente)
        ▼
Prometheus faz GET http://<pod-ip>:<porta><path> a cada interval
        │
        ▼
Métricas chegam com labels do Kubernetes adicionados automaticamente:
  pod, namespace, service, endpoint, job, instance
```

### Ciclo de vida de um alerta (PrometheusRule)

```
PrometheusRule criado → Operator carrega as regras no Prometheus
        │
        ▼ a cada evaluationInterval (padrão: 15s ou configurado por grupo)
Prometheus avalia a expressão `expr`
        │
        ├── false → estado INACTIVE
        │
        └── true → estado PENDING
                    │  aguarda `for:` (ex: 5m)
                    ├── condição verdadeira por mais de `for:` → FIRING
                    │     Prometheus envia para AlertManager
                    └── condição falsa antes de completar `for:` → volta a INACTIVE
```

A métrica `ALERTS{alertstate="firing"}` fica visível no Prometheus para cada alerta disparado.

---

## Na prática — ServiceMonitor

### Estrutura completa de um ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: minha-app
  namespace: estudo                    # namespace do ServiceMonitor
  labels:
    release: kube-prometheus-stack     # só necessário se o Operator usar selector de label
spec:
  # Quais Services monitorar
  selector:
    matchLabels:
      app: minha-app                   # label no Service
    matchExpressions:
    - key: env
      operator: In
      values: [producao, staging]

  # Em quais namespaces procurar os Services
  namespaceSelector:
    matchNames:
    - estudo
    - producao
    # any: true  ← para buscar em todos os namespaces

  # Como fazer o scrape (um item por porta/endpoint)
  endpoints:
  - port: http-metrics                 # nome da porta no Service (não o número)
    path: /metrics                     # padrão se omitido
    interval: 30s                      # padrão: herda do Prometheus (15s)
    scrapeTimeout: 10s
    scheme: http                       # http ou https

    # Adicionar labels extras nas métricas coletadas
    relabelings:
    - sourceLabels: [__meta_kubernetes_pod_node_name]
      targetLabel: node
    - sourceLabels: [__meta_kubernetes_namespace]
      targetLabel: kubernetes_namespace

    # Filtrar métricas APÓS a coleta (economizar storage)
    metricRelabelings:
    - sourceLabels: [__name__]
      regex: 'go_.*'                   # dropar todas as métricas Go internas
      action: drop

    # Honrar labels das métricas (cuidado: pode sobrescrever labels do Prometheus)
    honorLabels: false                 # padrão — Prometheus tem prioridade

    # Autenticação (se o /metrics requer auth)
    bearerTokenSecret:
      name: app-metrics-token
      key: token
    tlsConfig:
      insecureSkipVerify: true        # para certificados self-signed internos
```

### O Service precisa de porta nomeada

```yaml
# ERRADO — porta sem nome
ports:
- port: 8080
  targetPort: 8080

# CORRETO — porta nomeada (ServiceMonitor referencia pelo nome)
ports:
- name: http-metrics
  port: 8080
  targetPort: 8080
```

### ServiceMonitor para múltiplas portas

```yaml
endpoints:
- port: http          # porta da aplicação (para black-box monitoring)
  path: /healthz
  interval: 60s

- port: http-metrics  # porta de métricas Prometheus
  path: /metrics
  interval: 15s
```

### ServiceMonitor com TLS (aplicação expõe /metrics em HTTPS)

```yaml
endpoints:
- port: https-metrics
  scheme: https
  tlsConfig:
    ca:
      secret:
        name: app-ca-cert
        key: ca.crt
    cert:
      secret:
        name: app-client-cert
        key: tls.crt
    keySecret:
      name: app-client-cert
      key: tls.key
```

### Verificar se o ServiceMonitor foi carregado

```bash
# No Prometheus UI: Status → Service Discovery
# Procurar pelo nome do ServiceMonitor

# Via kubectl — ver a config gerada
kubectl exec -n monitoring <prometheus-pod> -- \
  cat /etc/prometheus/config_out/prometheus.env.yaml | grep -A20 "estudo/minha-app"

# Verificar os targets ativos
kubectl exec -n monitoring <prometheus-pod> -- \
  wget -qO- "localhost:9090/api/v1/targets" | jq '.data.activeTargets[] | select(.labels.job | contains("minha-app"))'
```

---

## Na prática — PodMonitor

### Quando usar PodMonitor em vez de ServiceMonitor

| Use ServiceMonitor quando | Use PodMonitor quando |
|---|---|
| A aplicação tem um Service | Não há Service (Jobs, CronJobs, bare Pods) |
| Quer balancear entre réplicas | Quer scrape individual de cada Pod |
| Caso padrão para aplicações web | Aplicações com métricas diferentes por réplica |

### Estrutura de um PodMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: minha-app-pods
  namespace: estudo
  labels:
    release: kube-prometheus-stack
spec:
  # Quais Pods monitorar (direto nos Pods, sem passar pelo Service)
  selector:
    matchLabels:
      app: minha-app

  namespaceSelector:
    matchNames:
    - estudo

  # Como fazer o scrape em cada Pod
  podMetricsEndpoints:
  - port: http-metrics               # nome da containerPort no Pod
    path: /metrics
    interval: 30s

    # Adicionar labels do Pod nas métricas
    relabelings:
    - sourceLabels: [__meta_kubernetes_pod_label_version]
      targetLabel: version           # adiciona o label "version" do Pod nas métricas

  # Labels do Pod para incluir nas métricas automaticamente
  podTargetLabels:
  - version
  - env
```

### Container precisa de containerPort nomeada

```yaml
# No Pod/Deployment:
containers:
- name: app
  ports:
  - name: http-metrics     # PodMonitor referencia este nome
    containerPort: 8080
```

---

## Na prática — PrometheusRule

### Estrutura completa

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: minha-app-rules
  namespace: estudo
  labels:
    release: kube-prometheus-stack   # necessário se o Operator filtrar por label
spec:
  groups:
  - name: minha-app.alerts           # nome do grupo (aparece na UI)
    interval: 30s                    # frequência de avaliação (opcional — herda do Prometheus)
    rules:

    # ── Regras de Alerta ──────────────────────────────────────────

    - alert: PodNaoReady
      expr: |
        kube_pod_status_ready{
          namespace="estudo",
          condition="false"
        } == 1
      for: 2m                        # deve ser true por 2min para disparar
      labels:
        severity: critical           # label usado pelo AlertManager para roteamento
        team: plataforma
      annotations:
        summary: "Pod {{ $labels.pod }} não está pronto"
        description: |
          Pod {{ $labels.pod }} no namespace {{ $labels.namespace }}
          está não-ready por mais de 2 minutos.
          Valor atual: {{ $value }}
        runbook: "https://wiki.staypuff.info/runbooks/pod-not-ready"
        dashboard: "https://grafana.staypuff.info/d/estudo-obs"

    - alert: AltaLatencia
      expr: |
        histogram_quantile(0.99,
          sum by (le, namespace, pod) (
            rate(http_request_duration_seconds_bucket{namespace="estudo"}[5m])
          )
        ) > 0.5
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Latência P99 acima de 500ms em {{ $labels.pod }}"
        description: "Latência P99 atual: {{ $value | humanizeDuration }}"

    - alert: MetricaAusente
      expr: absent(up{job=~"estudo/.*"})
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Nenhum target do namespace estudo está sendo coletado"
        description: "O ServiceMonitor do namespace estudo parou de funcionar."

  - name: minha-app.recording        # grupo separado para recording rules
    interval: 1m
    rules:

    # ── Recording Rules ───────────────────────────────────────────

    - record: namespace_job:http_requests_total:rate5m
      expr: |
        sum by (namespace, job) (
          rate(http_requests_total[5m])
        )

    - record: namespace_pod:container_cpu_usage:rate5m
      expr: |
        sum by (namespace, pod) (
          rate(container_cpu_usage_seconds_total{container!=""}[5m])
        )
```

### Convenção de nomes para recording rules

O padrão oficial do Prometheus é:
```
<labels_agregados>:<métrica_base>:<operação>
```

Exemplos:
```
job:http_requests_total:rate5m
namespace_pod:container_memory_bytes:avg
instance_path:http_request_duration_seconds:p99
```

### Funções de template em annotations

```yaml
annotations:
  # Valor da métrica formatado
  summary: "Uso de CPU: {{ $value | humanizePercentage }}"
  # humanize: 1234567 → "1.234M"
  # humanizeDuration: 90 → "1m30s"
  # humanizePercentage: 0.87 → "87%"
  # humanize1024: bytes → "1.17GB"

  # Labels do alerta
  description: "Pod {{ $labels.pod }} no namespace {{ $labels.namespace }}"

  # Iterar sobre múltiplos alertas no mesmo grupo (no AlertManager)
  message: |
    {{ range .Alerts }}
    • {{ .Annotations.summary }}
    {{ end }}
```

### Testar regras antes de aplicar (promtool)

```bash
# Validar sintaxe do YAML e expressões PromQL
promtool check rules minha-app-rules.yaml

# Testar uma expressão PromQL diretamente no Prometheus
kubectl exec -n monitoring <prometheus-pod> -- \
  promtool query instant http://localhost:9090 \
  'kube_pod_status_ready{namespace="estudo", condition="false"} == 1'

# Verificar alertas carregados
kubectl exec -n monitoring <prometheus-pod> -- \
  wget -qO- "localhost:9090/api/v1/rules" | jq '.data.groups[] | select(.name | contains("minha-app"))'
```

---

## Casos de uso e boas práticas

### Estruture as regras por severidade e ação esperada

| Severity | Significado | `for:` típico | Destino |
|---|---|---|---|
| `critical` | Impacto imediato no usuário, precisa de ação agora | 1–5min | PagerDuty/Telegram urgente |
| `warning` | Degradação em progresso, precisa de atenção | 10–15min | Slack/Telegram normal |
| `info` | Informativo, sem ação urgente | 30min+ | Canal de logs, ticket |

### Recording rules para queries pesadas

Qualquer query com `histogram_quantile` + `rate` sobre janelas longas (> 5min) deve ser pré-computada:

```yaml
# Sem recording rule: executada em tempo real por cada dashboard/alerta
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[30m]))

# Com recording rule: pré-computada a cada 1min
- record: job:http_request_duration_seconds:p99_30m
  expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[30m]))
```

### Um PrometheusRule por aplicação/time

Evitar um PrometheusRule central para todo o cluster — dificulta ownership e aumenta o blast radius de um erro de sintaxe.

### `relabelings` vs `metricRelabelings` — a diferença crítica

```
Scrape iniciado
      │
      ▼
relabelings   → executado ANTES do scrape
                Modifica os targets (URL, labels de identificação)
                Pode dropar targets inteiros (action: drop no target)
      │
      ▼
HTTP GET /metrics
      │
      ▼
metricRelabelings → executado APÓS o scrape, em cada métrica individual
                    Pode dropar métricas específicas, renomear labels, etc.
```

Use `relabelings` para: adicionar labels de contexto do Kubernetes às métricas, filtrar quais Pods são scrapeados.

Use `metricRelabelings` para: dropar métricas desnecessárias (economizar storage), renomear labels de métricas individuais.

### Evitar cardinalidade alta

```yaml
# PERIGOSO — user_id como label cria uma série por usuário (milhões)
http_requests_total{method="GET", user_id="12345"}

# CORRETO — usar histograma para distribuição
http_request_duration_seconds_bucket{method="GET", le="0.5"}
```

Se uma métrica com alta cardinalidade aparecer, use `metricRelabelings` para dropar o label ofensivo:

```yaml
metricRelabelings:
- sourceLabels: [user_id]
  regex: '.*'
  targetLabel: user_id
  replacement: ''              # zerar o valor do label (remove sem dropar a métrica)
```

---

## Troubleshooting — cenários reais de produção

### ServiceMonitor criado mas target não aparece no Prometheus

**Diagnóstico em ordem:**

```bash
# 1. O ServiceMonitor existe?
kubectl get servicemonitor -n estudo

# 2. O Prometheus Operator carregou o ServiceMonitor?
kubectl logs -n monitoring -l app=prometheus-operator | grep -i "servicemonitor\|estudo"

# 3. A config foi gerada?
kubectl exec -n monitoring <prometheus-pod> -- \
  cat /etc/prometheus/config_out/prometheus.env.yaml | grep "estudo"
# Se não aparece: problema no selector do Operator ou label faltando

# 4. O Service existe com o label correto?
kubectl get svc -n estudo -l app=minha-app

# 5. A porta do Service tem nome?
kubectl get svc minha-app-svc -n estudo -o jsonpath='{.spec.ports}'

# 6. Há Endpoints no Service?
kubectl get endpoints minha-app-svc -n estudo
```

### Target UP mas métricas não chegam

```bash
# Testar o endpoint de métricas diretamente
kubectl exec -n monitoring <prometheus-pod> -- \
  wget -qO- http://<pod-ip>:8080/metrics | head -20

# Verificar se há NetworkPolicy bloqueando
kubectl get networkpolicy -n estudo
# Se existir: o Prometheus precisa ter ingress liberado da porta de métricas

# Verificar o scrape timeout (se as métricas demoram para serem geradas)
# Aumentar no ServiceMonitor:
# scrapeTimeout: 30s
```

### PrometheusRule com erro de sintaxe — alerta não dispara

```bash
# Verificar logs do Operator
kubectl logs -n monitoring -l app=prometheus-operator | grep -i "error\|invalid\|rule"

# Verificar o estado das regras no Prometheus
# Status → Rules: regras com erro aparecem em vermelho

# Validar localmente antes de aplicar
promtool check rules regras.yaml
# output: Checking regras.yaml
#           SUCCESS: 3 rules found
```

### Alerta em `Pending` que nunca vira `Firing`

```promql
# Ver o valor atual da expressão do alerta
kube_pod_status_ready{namespace="estudo", condition="false"} == 1

# Se retorna vazio → condição nunca foi verdadeira
# Verificar se os labels estão corretos
kube_pod_status_ready{namespace="estudo"}
# Quais são os valores de "condition"? true, false, ou unknown?
```

### Alerta disparando com falsos positivos

```bash
# Aumentar o `for:` para filtrar flaps curtos
# Verificar se o `scrapeInterval` é maior que o `for:` (impossível de atingir)
# ex: scrapeInterval: 30s e for: 15s → nunca vai disparar (avalia a cada 30s)
```

---

## Nível avançado — relabelings, multi-cluster e SLO

### relabelings para enriquecer métricas com dados do Kubernetes

```yaml
relabelings:
# Adicionar o nome do node ao target
- sourceLabels: [__meta_kubernetes_pod_node_name]
  targetLabel: node

# Adicionar a versão da imagem como label
- sourceLabels: [__meta_kubernetes_pod_container_image]
  regex: '.*:(.*)'              # extrair só a tag da imagem
  targetLabel: image_version
  replacement: '$1'

# Renomear o label "instance" para mostrar o pod name
- sourceLabels: [__meta_kubernetes_pod_name]
  targetLabel: instance

# Dropar Pods com label "monitoring=false"
- sourceLabels: [__meta_kubernetes_pod_label_monitoring]
  regex: 'false'
  action: drop
```

### metricRelabelings para filtrar e transformar

```yaml
metricRelabelings:
# Dropar todas as métricas Go internas (economiza ~30% do storage)
- sourceLabels: [__name__]
  regex: 'go_(gc|goroutines|memstats|threads).*'
  action: drop

# Dropar métricas de alta cardinalidade
- sourceLabels: [__name__, endpoint]
  regex: 'http_requests_total;/healthz'  # não armazenar healthcheck calls
  action: drop

# Normalizar um label
- sourceLabels: [method]
  regex: '(get|GET)'
  targetLabel: method
  replacement: 'GET'
```

### PrometheusRule de SLO com burn rate

Alertas de SLO baseados em taxa de queima do error budget (mais eficazes que threshold simples):

```yaml
groups:
- name: slo.webapp
  rules:
  # Taxa de erro atual (janela 5min para alertas rápidos)
  - record: job:http_error_rate:rate5m
    expr: |
      sum(rate(http_requests_total{status=~"5..", namespace="estudo"}[5m]))
      / sum(rate(http_requests_total{namespace="estudo"}[5m]))

  # SLO: 99.9% de disponibilidade
  # Burn rate > 14.4x em 1h → error budget acaba em 3 dias
  - alert: SLO_BurnRateFast
    expr: job:http_error_rate:rate5m > (14.4 * 0.001)
    for: 2m
    labels:
      severity: critical
      slo: webapp-availability
    annotations:
      summary: "SLO em risco — burn rate alto"
      description: |
        Taxa de erro atual {{ $value | humanizePercentage }}.
        SLO de 99.9% será violado em menos de 3 dias no ritmo atual.
```

### ScrapeConfig — configuração de scrape sem ServiceMonitor (novo CRD)

O Prometheus Operator v0.65+ introduziu o CRD `ScrapeConfig` para casos avançados não cobertos pelo ServiceMonitor:

```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: ScrapeConfig
metadata:
  name: external-service
  namespace: monitoring
spec:
  staticConfigs:
  - labels:
      job: nfs-server
    targets:
    - 192.168.3.11:9100        # node-exporter no NFS server (fora do K8s)
```

---

## Referências

- [Prometheus Operator — API Reference](https://prometheus-operator.dev/docs/operator/api/)
- [ServiceMonitor spec](https://prometheus-operator.dev/docs/operator/api/#monitoring.coreos.com/v1.ServiceMonitor)
- [PodMonitor spec](https://prometheus-operator.dev/docs/operator/api/#monitoring.coreos.com/v1.PodMonitor)
- [PrometheusRule spec](https://prometheus-operator.dev/docs/operator/api/#monitoring.coreos.com/v1.PrometheusRule)
- [Prometheus Relabeling](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config)
- [Alerting Rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [promtool](https://prometheus.io/docs/prometheus/latest/command-line/promtool/)
- Relacionados: [[06-Observabilidade/kube-prometheus-stack]] | [[05-Kubernetes/labels-annotations]]
