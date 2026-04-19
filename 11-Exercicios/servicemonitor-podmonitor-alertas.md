---
tags:
  - exercicios
  - observabilidade
  - servicemonitor
  - podmonitor
  - prometheusrule
tipo: exercicios
area: observabilidade
conteudo: "[[06-Observabilidade/servicemonitor-podmonitor-alertas]]"
trilha: "[[00-Trilha/observabilidade]]"
---

# Exercícios — ServiceMonitor, PodMonitor e PrometheusRule

> Conteúdo: [[06-Observabilidade/servicemonitor-podmonitor-alertas]] | Trilha: [[00-Trilha/observabilidade]]

> [!warning] Restrições de Produção
> Namespaces `controle-gastos`, `promobot` e `databases` → **somente leitura** (inventário, inspeção).
> Toda criação de recursos ocorre no namespace `estudo`.

---

## Exercício 1: Inventário de ServiceMonitors no Cluster

**Contexto:** Você acaba de entrar em um time que tem o kube-prometheus-stack rodando há meses. Não existe documentação sobre o que está sendo monitorado. Sua primeira tarefa é mapear todos os ServiceMonitors existentes e entender qual Prometheus os está coletando.

**Missão:** Produzir um inventário completo dos ServiceMonitors e confirmar que o Prometheus os está descobrindo corretamente.

**Requisitos:**
- [ ] Listar todos os ServiceMonitors em todos os namespaces
- [ ] Para cada ServiceMonitor, identificar: namespace, nome, o `selector.matchLabels` e o `namespaceSelector`
- [ ] No Prometheus UI (`prometheus.staypuff.info`), verificar em **Status → Service Discovery** quantos targets cada ServiceMonitor gerou
- [ ] Identificar o label obrigatório que o Prometheus usa para descobrir ServiceMonitors (`spec.serviceMonitorSelector` no CRD Prometheus)
- [ ] Listar os namespaces que o Prometheus está autorizado a acessar (`spec.serviceMonitorNamespaceSelector`)

**Verificação:**
```bash
# Deve listar todos os ServiceMonitors com suas informações
sudo kubectl get servicemonitor -A -o wide

# Deve mostrar o seletor de ServiceMonitors do Prometheus
sudo kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}' | jq .
```

---

## Exercício 2: ServiceMonitor para Aplicação Custom

**Contexto:** O time de produto deployou uma aplicação Python que expõe métricas no endpoint `/metrics` na porta `8080`. O container tem label `app: metricas-app`. Nenhum ServiceMonitor foi criado — as métricas não aparecem no Prometheus.

**Missão:** Criar a aplicação de teste e o ServiceMonitor correto para que ela apareça como target `UP` no Prometheus.

**Requisitos:**
- [ ] Criar namespace `estudo` (se não existir)
- [ ] Criar um Deployment com a imagem `prom/prometheus` (ela expõe `/metrics`) com label `app: metricas-app` no namespace `estudo`
- [ ] Criar um Service do tipo ClusterIP apontando para o Deployment com uma porta **nomeada** (ex: `name: http-metrics`)
- [ ] Criar um ServiceMonitor no namespace `estudo` que:
  - Selecione o Service pelo label `app: metricas-app`
  - Referencie a porta pelo nome (não pelo número)
  - Tenha o label `release: kube-prometheus-stack` (obrigatório para descoberta)
- [ ] Confirmar no Prometheus UI que o target aparece e está `UP`

**Verificação:**
```bash
# O ServiceMonitor deve existir
sudo kubectl get servicemonitor -n estudo

# O target deve aparecer em Status → Targets no Prometheus UI
# Job name gerado: estudo/metricas-app/<index>
```

> [!tip] Porta Nomeada
> O campo `endpoints[].port` no ServiceMonitor aceita apenas nome — não número. O Service precisa ter a porta com `name:` definido.

---

## Exercício 3: Debug — ServiceMonitor Não Aparece nos Targets

**Contexto:** Um colega criou um ServiceMonitor, mas após 10 minutos o target não aparece no Prometheus. Ele jurou que está tudo certo. Você precisa diagnosticar.

**Missão:** Reproduzir o problema com um ServiceMonitor propositalmente errado e corrigi-lo.

**Requisitos:**
- [ ] Criar um ServiceMonitor com **pelo menos dois dos seguintes problemas**:
  1. Label `release: wrong-label` (não coincide com o serviceMonitorSelector do Prometheus)
  2. `namespaceSelector` excluindo o namespace onde o Service está
  3. `selector.matchLabels` com label que não existe no Service
  4. Porta referenciada por nome que não existe no Service
- [ ] Usando apenas `kubectl` e o Prometheus UI (**Status → Configuration** e **Status → Service Discovery**), identificar qual dos problemas está impedindo a descoberta
- [ ] Corrigir um problema por vez e verificar o efeito no Prometheus UI
- [ ] Documentar (em comentário no YAML ou em um arquivo de notas) qual foi o problema raiz

**Verificação:**
```bash
# Verificar se o Prometheus tem o ServiceMonitor na configuração gerada
# Acessar: prometheus.staypuff.info/api/v1/targets e filtrar pelo job

# Verificar events do Prometheus Operator
sudo kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus-operator --tail=50
```

> [!warning] Sem kubectl describe no ServiceMonitor
> O Prometheus Operator não escreve events no ServiceMonitor. O diagnóstico é feito nos logs do Operator e no Prometheus UI — não existe feedback direto no objeto.

---

## Exercício 4: PodMonitor para DaemonSet sem Service

**Contexto:** O node-exporter roda como DaemonSet e já tem um ServiceMonitor. Agora você precisa monitorar um job batch que roda como Pod esporádico — sem Service permanente — e expõe métricas na porta `9090`.

**Missão:** Criar um PodMonitor para coletar métricas diretamente de Pods sem Service.

**Requisitos:**
- [ ] Criar um Pod de teste no namespace `estudo` com label `app: batch-job` e a imagem `prom/prometheus` (expõe `/metrics` na porta 9090)
- [ ] Criar um PodMonitor que:
  - Selecione Pods pelo label `app: batch-job`
  - Use `podMetricsEndpoints` com a porta `9090`
  - Inclua o label `node` via `podTargetLabels` (copiar o label `kubernetes.io/hostname` do Pod para as métricas)
  - Tenha o label `release: kube-prometheus-stack`
- [ ] Verificar no Prometheus UI que o target aparece com o label `node` nas métricas coletadas
- [ ] Explicar por que não seria possível usar ServiceMonitor neste cenário

**Verificação:**
```bash
# PodMonitor deve existir
sudo kubectl get podmonitor -n estudo

# No Prometheus UI, em Status → Targets, o job do PodMonitor deve aparecer
# Métricas devem ter label node=<valor> adicionado pelo podTargetLabels
```

---

## Exercício 5: PrometheusRule — Alerta com Ciclo Pending → Firing

**Contexto:** O time quer um alerta quando qualquer Pod no namespace `estudo` ficar em estado `Pending` por mais de 2 minutos. Isso indica problema de scheduling, PVC não bound ou imagem inexistente.

**Missão:** Criar a PrometheusRule, forçar o cenário e acompanhar o alerta evoluir até `Firing`.

**Requisitos:**
- [ ] Criar uma PrometheusRule no namespace `estudo` com:
  - Grupo `estudo.pods`
  - Alerta `PodStuckPending` com `for: 2m`
  - Labels de severidade e roteamento (`severity: warning`, `team: estudo`)
  - Annotation `summary` e `description` usando template `{{ $labels.pod }}`
  - A expressão PromQL deve usar `kube_pod_status_phase`
- [ ] Verificar que a PrometheusRule aparece em **Status → Rules** no Prometheus UI
- [ ] Forçar o cenário: criar um Pod com imagem inexistente (`busybox:naoexiste`) ou com `nodeSelector` impossível
- [ ] Acompanhar em **Alerts** no Prometheus UI: `Inactive → Pending → Firing`
- [ ] Após o alerta Firing, deletar o Pod e confirmar que o alerta volta para `Inactive`

**Verificação:**
```bash
# PrometheusRule deve existir e ter o label correto
sudo kubectl get prometheusrule -n estudo -o yaml

# No Prometheus UI: Status → Rules → grupo estudo.pods
# Em Alerts: PodStuckPending deve aparecer após 2 minutos
```

> [!tip] Label Obrigatório
> A PrometheusRule precisa ter o label que o Prometheus usa para descoberta de regras (`spec.ruleSelector`). Inspecione o CRD Prometheus para verificar qual label é exigido no seu cluster.

---

## Exercício 6: Recording Rules — Aceleração de Queries

**Contexto:** O dashboard de CPU do Grafana está lento porque a query `rate(container_cpu_usage_seconds_total[5m])` precisa processar 48h de dados com alta cardinalidade. Recording rules pré-computam esse resultado a cada `scrape_interval`.

**Missão:** Criar recording rules para as métricas mais consultadas e medir a diferença de performance.

**Requisitos:**
- [ ] Criar uma PrometheusRule com grupo `estudo.recording` contendo ao menos 3 recording rules seguindo a convenção de nomes `level:metric:operations`:
  1. CPU por namespace: `namespace:container_cpu_usage_seconds_total:sum_rate`
  2. Memory por namespace: `namespace:container_memory_working_set_bytes:sum`
  3. Request rate por serviço: `job:http_requests_total:rate5m` (use qualquer métrica HTTP disponível)
- [ ] Verificar em **Status → Rules** que as recording rules estão avaliando (coluna "Last evaluation")
- [ ] No Prometheus UI, comparar tempo de execução:
  - Query original com `rate(container_cpu_usage_seconds_total[5m])` agrupada por namespace
  - Query usando a recording rule `namespace:container_cpu_usage_seconds_total:sum_rate`
- [ ] Confirmar que ambas retornam valores equivalentes

**Verificação:**
```bash
# As recording rules devem aparecer como métricas no Prometheus
# curl prometheus.staypuff.info/api/v1/label/__name__/values | jq '.data[]' | grep namespace:
```

---

## Exercício 7: relabelings e metricRelabelings

**Contexto:** O time de segurança identificou que algumas métricas expõem IPs internos em labels de alta cardinalidade (`ip=192.168.x.x`), causando explosão de séries no TSDB. Além disso, você quer adicionar o label `cluster=homelab` em todas as métricas de um ServiceMonitor específico.

**Missão:** Usar `relabelings` para enriquecer targets e `metricRelabelings` para filtrar métricas problemáticas.

**Requisitos:**
- [ ] No ServiceMonitor criado no Exercício 2, adicionar via `relabelings`:
  - Label `cluster` com valor fixo `homelab` (usando `replacement: homelab` e `targetLabel: cluster`)
  - Preservar o label `__meta_kubernetes_pod_node_name` como `node`
- [ ] Adicionar via `metricRelabelings` no mesmo ServiceMonitor:
  - Dropar todas as métricas que contenham `go_` no nome (reduzir cardinalidade das métricas de runtime Go)
  - Usar `action: drop` com `sourceLabels: [__name__]`
- [ ] Verificar no Prometheus UI que:
  - Targets têm o label `cluster=homelab`
  - Métricas `go_*` não aparecem mais para esse job
  - Outras métricas continuam sendo coletadas

**Verificação:**
```bash
# No Prometheus UI:
# {job="estudo/metricas-app/0", cluster="homelab"} deve retornar métricas
# go_goroutines{job="estudo/metricas-app/0"} deve retornar "no data"
```

> [!info] relabelings vs metricRelabelings
> `relabelings` roda **antes** do scrape — modifica targets e labels de descoberta.
> `metricRelabelings` roda **depois** do scrape — modifica ou filtra métricas individuais.
> Dropar métricas em `metricRelabelings` ainda consome banda de rede — o exporter continua produzindo, o Prometheus só ignora após receber.

---

## Exercício 8: Pipeline Completo — Monitoramento de Nova Aplicação (Staff-level)

**Contexto:** O time vai fazer o deploy de uma nova aplicação no namespace `estudo`. Você é o SRE responsável por instrumentar o monitoramento completo antes do go-live: discovery, alertas e roteamento para o canal correto.

**Missão:** Criar o pipeline completo de observabilidade para a aplicação `api-estudo`.

**Requisitos:**

**Parte A — Aplicação e ServiceMonitor:**
- [ ] Criar Deployment `api-estudo` com 2 réplicas usando `prom/prometheus` como imagem (simula uma API com métricas), labels `app: api-estudo, team: estudo`
- [ ] Criar Service com porta nomeada `http-metrics: 9090`
- [ ] Criar ServiceMonitor com:
  - `release: kube-prometheus-stack`
  - `relabelings` adicionando `cluster=homelab` e `team=estudo`
  - `metricRelabelings` dropando métricas com prefixo `go_`
  - `scrapeInterval: 30s` (diferente do global para demonstrar override)

**Parte B — PrometheusRule:**
- [ ] Criar PrometheusRule com grupo `api-estudo.alerts` contendo:
  - Alerta `ApiEstudoDown`: dispara quando `up{job=~"estudo/api-estudo.*"} == 0` por 1 minuto
  - Alerta `ApiEstudoHighMemory`: dispara quando memory do Pod > 100Mi por 5 minutos
  - Recording rule `job:api_estudo_up:avg` com a média de targets UP
- [ ] Forçar o alerta `ApiEstudoDown` deletando o Deployment e aguardar Firing

**Parte C — AlertManager (inspeção):**
- [ ] No AlertManager UI (`alertmanager.staypuff.info`), verificar se o alerta `ApiEstudoDown` chegou
- [ ] Identificar para qual receiver o alerta foi roteado (qual rota no `alertmanager.yaml` foi correspondida)
- [ ] Criar um silence de 5 minutos para o alerta usando a UI ou a API do AlertManager
- [ ] Confirmar que o alerta ainda aparece como `Firing` no Prometheus mas está silenciado no AlertManager

**Verificação:**
```bash
# ServiceMonitor, PrometheusRule e recursos da aplicação
sudo kubectl get servicemonitor,prometheusrule,deploy,svc -n estudo

# Verificar que o alerta foi para Firing
# Acessar: prometheus.staypuff.info → Alerts → ApiEstudoDown

# Verificar silence no AlertManager
# curl -s alertmanager.staypuff.info/api/v2/silences | jq '.[] | select(.status.state=="active")'
```

> [!warning] Cleanup
> Ao finalizar, deletar todos os recursos criados nos exercícios 2-8 no namespace `estudo` para liberar recursos do cluster.
> `sudo kubectl delete ns estudo && sudo kubectl create ns estudo`
