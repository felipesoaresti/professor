---
tags:
  - trilha
  - observabilidade
area: observabilidade
tipo: trilha
---

# Trilha Observabilidade

> Canvas geral: [[trilha-devops-sre]]

## Assuntos Cobertos

| Assunto | Conteúdo | Exercícios | Status |
|---|---|---|---|
| kube-prometheus-stack — Prometheus, Grafana, AlertManager | [[06-Observabilidade/kube-prometheus-stack]] | [[11-Exercicios/kube-prometheus-stack]] | Exercícios gerados — pendente execução |
| ServiceMonitor, PodMonitor e PrometheusRule | [[06-Observabilidade/servicemonitor-podmonitor-alertas]] | [[11-Exercicios/servicemonitor-podmonitor-alertas]] | Exercícios gerados — pendente execução |

## Sequência Recomendada

### Coleta e Armazenamento
1. [[06-Observabilidade/kube-prometheus-stack]] — Prometheus Operator, ServiceMonitor, PodMonitor, PromQL, TSDB ✅
2. [[06-Observabilidade/servicemonitor-podmonitor-alertas]] — ServiceMonitor, PodMonitor, PrometheusRule, relabelings, SLO burn rate ✅

### Visualização
2. Grafana avançado — variáveis de template, transformações, alertas nativos, Loki data source *(a fazer)*

### Alertas e Incidentes
3. AlertManager avançado — inibição, rotas complexas, silences programáticos, API *(a fazer)*
4. SLO e Error Budget — Sloth, Pyrra, PrometheusRules de SLO, burn rate alerts *(a fazer)*

### Logs
5. Loki + Promtail — log aggregation, LogQL, correlação métricas/logs no Grafana *(a fazer)*

### Tracing
6. Tempo + OpenTelemetry — distributed tracing, instrumentação, correlação traces/métricas/logs *(a fazer)*

### Longo Prazo
7. Thanos ou VictoriaMetrics — retenção longa, federação multi-cluster, downsampling *(a fazer)*

## Próximo Sugerido

**Grafana avançado** — Variáveis de template (dropdowns dinâmicos), transformações de dados, Loki data source para logs, e alertas nativos do Grafana (Grafana Alerting, que é diferente do AlertManager). O stack instalado já tem tudo — explorar além dos dashboards básicos.

Ou: **Loki + Promtail** — Os logs dos Pods estão acessíveis via `kubectl logs`, mas sem histórico e sem busca. O Loki agrega logs de todos os Pods com os mesmos labels do Prometheus, permitindo correlação direta no Grafana.

Ou: **SLO e Error Budget** — Aprofundar os conceitos de burn rate que já aparecem nas PrometheusRules. Ferramentas como Sloth ou Pyrra geram as regras automaticamente a partir de uma definição de SLO simples.
