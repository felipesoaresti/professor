---
tags:
  - exercicios
  - kubernetes
  - probes
  - readiness
  - liveness
  - startup
tipo: exercicios
area: kubernetes
conteudo: "[[05-Kubernetes/probes]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Exercícios — Probes em Deployments

> Conteúdo: [[05-Kubernetes/probes]] | Trilha: [[00-Trilha/kubernetes]]

**Infraestrutura:** k8s-master (192.168.3.30), worker1 (.31), worker2 (.32)
**Namespace de trabalho:** `default` ou `estudo-felipe`
**Atenção:** Não tocar em `controle-gastos`, `promobot`, `databases/postgres`.

---

## Exercício 1: Observar o comportamento sem probes

**Contexto:** Antes de configurar probes, você precisa entender o problema que elas resolvem. Você vai ver na prática o que acontece quando o Kubernetes manda tráfego para um Pod que ainda não está pronto.

**Missão:** Crie um Deployment sem nenhuma probe e observe o comportamento durante o deploy. Use uma imagem que simula inicialização lenta.

**Requisitos:**
- [ ] Criar Deployment `webapp-noprobe` com 3 réplicas e `nginx:1.27`, sem nenhuma probe configurada
- [ ] Criar um Service ClusterIP apontando para o Deployment
- [ ] Atualizar a imagem para `nginx:1.26` e observar com `kubectl get pods -w`
- [ ] Verificar em qual momento exato os Pods novos aparecem nos Endpoints do Service
- [ ] Confirmar que os Pods entram nos Endpoints **antes** de qualquer verificação de saúde
- [ ] Registrar: qual é o tempo entre `Running` e aparecer nos Endpoints?
- [ ] Deletar os recursos ao final

**Verificação:**
```bash
kubectl get endpoints webapp-noprobe -w
# Observar: o IP do Pod novo aparece imediatamente quando o container inicia,
# sem esperar nenhuma verificação
```

---

## Exercício 2: readinessProbe — controlando o tráfego durante deploys

**Contexto:** O time reportou 502s durante os últimos 3 deploys. A causa é tráfego chegando antes da app estar pronta. Você vai corrigir configurando uma readinessProbe.

**Missão:** Configure uma readinessProbe httpGet em um Deployment nginx e confirme que o Pod só entra nos Endpoints após passar na probe.

**Requisitos:**
- [ ] Criar Deployment `webapp` com 3 réplicas e `nginx:1.27`
- [ ] Configurar readinessProbe com `httpGet path: / port: 80`, `periodSeconds: 5`, `failureThreshold: 3`
- [ ] Criar Service ClusterIP para o Deployment
- [ ] Observar com dois terminais: `kubectl get pods -w` e `kubectl get endpoints webapp -w`
- [ ] Confirmar que o Pod só aparece nos Endpoints **após** a readinessProbe passar (status `1/1 Running`)
- [ ] Forçar a readiness a falhar: adicionar um `command` que torna o nginx não disponível na porta, ou usar `exec` para criar um arquivo de flag
- [ ] Verificar que o Pod sai dos Endpoints quando a readiness falha (mas não reinicia)
- [ ] Deletar os recursos ao final

**Verificação:**
```bash
kubectl describe pod <nome> | grep -E "Readiness|Ready"
kubectl get endpoints webapp
# Pod com READY 0/1 não deve aparecer nos Endpoints
# Pod com READY 1/1 deve aparecer nos Endpoints
```

---

## Exercício 3: livenessProbe — detectando e reiniciando containers travados

**Contexto:** Uma aplicação tem um bug que causa travamento após 2 minutos de execução — o processo continua vivo mas para de responder. O SRE precisa que o Kubernetes detecte isso e reinicie automaticamente.

**Missão:** Simule um container que "trava" e configure uma livenessProbe que detecte o travamento e force o reinício.

**Requisitos:**
- [ ] Criar Pod (não Deployment) com a seguinte spec de container:
  ```yaml
  image: busybox
  command: ["/bin/sh", "-c"]
  args:
    - |
      # Simula app que trava após 60s
      touch /tmp/healthy
      sleep 60
      rm /tmp/healthy
      sleep 3600
  ```
- [ ] Configurar livenessProbe exec que verifica se `/tmp/healthy` existe: `test -f /tmp/healthy`
- [ ] Configurar com `periodSeconds: 10`, `failureThreshold: 3` (total: 30s para detectar)
- [ ] Observar com `kubectl get pod <nome> -w` o Pod passando de Running para reiniciando
- [ ] Confirmar com `kubectl describe pod <nome>` os eventos de "Liveness probe failed"
- [ ] Verificar a contagem de `RESTARTS` aumentando
- [ ] Deletar o Pod ao final

**Verificação:**
```bash
kubectl get pod <nome> -w
# Deve ver: RESTARTS aumentando após ~90s (60s de sleep + 30s de probe failures)

kubectl describe pod <nome> | grep -A5 Events
# Warning  Unhealthy  Liveness probe failed: ...
# Normal   Killing    Container <nome> failed liveness probe, will be restarted
```

---

## Exercício 4: startupProbe — protegendo apps com inicialização lenta

**Contexto:** O time de Java reclamou que o Kubernetes fica reiniciando os Pods da aplicação durante o boot da JVM. A livenessProbe está configurada com `initialDelaySeconds: 30` mas a JVM demora 90s. Você vai resolver com startupProbe.

**Missão:** Configure uma startupProbe que dá tempo suficiente para uma "app lenta" inicializar, sem comprometer a detecção de falhas pós-inicialização.

**Requisitos:**
- [ ] Criar Pod que simula inicialização lenta:
  ```yaml
  command: ["/bin/sh", "-c"]
  args:
    - |
      echo "Iniciando... aguarde"
      sleep 45   # simula JVM warmup
      echo "Pronto!"
      touch /tmp/started
      sleep 3600
  ```
- [ ] Configurar startupProbe exec que verifica `/tmp/started` com `failureThreshold: 10` e `periodSeconds: 10` (100s de tolerância total)
- [ ] Configurar livenessProbe exec que também verifica `/tmp/started` com `periodSeconds: 15`, `failureThreshold: 3`
- [ ] Observar que a livenessProbe **não** reinicia o Pod durante os primeiros 45s (startupProbe protege)
- [ ] Verificar via `kubectl describe pod` que a startupProbe está rodando antes da liveness
- [ ] Após a startupProbe passar, verificar que a livenessProbe assume
- [ ] Deletar o Pod ao final

**Verificação:**
```bash
kubectl describe pod <nome> | grep -E "Startup|Liveness"
# Startup Probe deve mostrar success após ~50s
# Liveness Probe deve mostrar success após a startup passar

kubectl get pod <nome> -w
# Pod NÃO deve reiniciar durante os primeiros 45s
# Pod deve ficar Running estável após a startup completar
```

---

## Exercício 5: As três probes juntas — configuração de referência

**Contexto:** Você vai configurar um Deployment seguindo o padrão de produção completo, com as três probes, preStop hook, e terminationGracePeriodSeconds. Esta é a configuração que você deve usar como template em workloads reais.

**Missão:** Crie um Deployment nginx com as três probes configuradas corretamente e valide o comportamento durante um RollingUpdate.

**Requisitos:**
- [ ] Criar Deployment `webapp-full` com 4 réplicas, `nginx:1.26`, e as três probes:
  - `startupProbe`: httpGet `/`, porta 80, `failureThreshold: 15`, `periodSeconds: 10`
  - `readinessProbe`: httpGet `/`, porta 80, `periodSeconds: 5`, `failureThreshold: 3`
  - `livenessProbe`: httpGet `/`, porta 80, `periodSeconds: 15`, `failureThreshold: 3`
- [ ] Adicionar `preStop: exec: command: ["sleep", "5"]`
- [ ] Configurar `terminationGracePeriodSeconds: 30`
- [ ] Configurar `maxUnavailable: 0` e `maxSurge: 1` na strategy
- [ ] Criar Service ClusterIP para o Deployment
- [ ] Executar um RollingUpdate para `nginx:1.27` e observar que:
  - Cada Pod novo passa pela startupProbe antes de ser adicionado aos Endpoints
  - O Service sempre tem ≥ 4 Endpoints durante o rollout
  - O rollout avança apenas quando o Pod novo está `1/1 Ready`
- [ ] Deletar os recursos ao final

**Verificação:**
```bash
# Em terminal separado, monitorar endpoints durante o rollout
kubectl get endpoints webapp-full -w

# O número de IPs nos Endpoints nunca deve cair abaixo de 4 (maxUnavailable: 0)
kubectl rollout status deployment/webapp-full
```

---

## Exercício 6: Diagnosticando CrashLoopBackOff causado por probe

**Contexto:** Você recebe um alerta às 3h: "webapp em CrashLoopBackOff". O deploy foi feito ontem e funcionou, mas agora os Pods estão reiniciando. Você precisa diagnosticar se é a livenessProbe ou um problema real na aplicação.

**Missão:** Simule um CrashLoopBackOff causado por livenessProbe mal configurada, faça o diagnóstico completo e corrija sem downtime.

**Requisitos:**
- [ ] Criar Deployment `webapp-crash` com `nginx:1.27` e a seguinte livenessProbe:
  ```yaml
  livenessProbe:
    httpGet:
      path: /pagina-que-nao-existe
      port: 80
    initialDelaySeconds: 5
    periodSeconds: 10
    failureThreshold: 3
  ```
- [ ] Observar o Pod entrando em CrashLoopBackOff em ~35s
- [ ] Executar o diagnóstico:
  - Verificar eventos do Pod com `kubectl describe`
  - Identificar que é "Liveness probe failed" e não um crash do container
  - Confirmar que o nginx está funcionando corretamente apesar do "crash"
- [ ] Corrigir a livenessProbe para usar `/` (que retorna 200 no nginx padrão)
- [ ] Confirmar que o Pod estabiliza após a correção
- [ ] Deletar o Deployment ao final

**Verificação:**
```bash
kubectl describe pod <nome> | grep -E "Warning|Liveness|Killing"
# Deve mostrar: Warning Unhealthy → Warning Killing → loop

# Após correção:
kubectl get pod <nome> -w
# Deve estabilizar: 1/1 Running, RESTARTS para de aumentar
```

---

## Exercício 7: Probe com tcpSocket e exec para workloads não-HTTP (avançado)

**Contexto:** O time vai deployar um Redis no cluster para uso dos serviços. Redis não tem endpoint HTTP — você precisa configurar probes apropriadas.

**Missão:** Crie um Pod Redis com livenessProbe via tcpSocket e readinessProbe via exec usando `redis-cli ping`.

**Requisitos:**
- [ ] Criar Pod `redis` com imagem `redis:7-alpine`
- [ ] Configurar livenessProbe tcpSocket na porta 6379, `periodSeconds: 10`, `failureThreshold: 3`
- [ ] Configurar readinessProbe exec: `redis-cli ping` (deve retornar `PONG`), `periodSeconds: 5`, `failureThreshold: 3`
- [ ] Confirmar que ambas as probes passam com `kubectl describe pod redis`
- [ ] Simular um "travamento" do Redis: dentro do container, executar `redis-cli debug sleep 60` (bloqueia o Redis por 60s)
- [ ] Observar qual probe detecta o problema primeiro (exec verifica resposta real, tcpSocket apenas conexão TCP)
- [ ] Registrar: a livenessProbe (tcpSocket) falha? Por quê ou por quê não?
- [ ] Deletar o Pod ao final

**Verificação:**
```bash
kubectl describe pod redis | grep -E "Liveness|Readiness"
# Ambas devem mostrar sucesso inicial

# Após debug sleep:
kubectl describe pod redis | grep -A5 Events
# readinessProbe exec deve falhar (redis-cli ping trava)
# livenessProbe tcpSocket pode não falhar (porta ainda responde TCP)
```

---

## Exercício 8: Otimizar probes para zero impacto no RollingUpdate (Staff-level)

**Contexto:** O time mediu que os deploys estão causando ~2s de aumento em error rate. A suspicita é que os Pods antigos estão sendo terminados enquanto ainda têm conexões ativas. Você vai configurar o shutdown gracioso completo.

**Missão:** Configure um Deployment com shutdown 100% gracioso: preStop hook, terminationGracePeriodSeconds adequado, e readinessProbe agressiva para remoção rápida dos Endpoints.

**Requisitos:**
- [ ] Criar Deployment `webapp-graceful` com 4 réplicas e `nginx:1.27`
- [ ] Configurar `preStop: exec: command: ["/bin/sh", "-c", "sleep 10"]` — simula drenagem de conexões
- [ ] Configurar `terminationGracePeriodSeconds: 60` — tempo total de shutdown
- [ ] Configurar readinessProbe com `periodSeconds: 2` e `failureThreshold: 1` — remove dos Endpoints rapidamente quando a probe falha
- [ ] Executar um RollingUpdate e monitorar:
  - Quanto tempo um Pod antigo permanece nos Endpoints após receber SIGTERM
  - Se o preStop hook roda antes do SIGTERM (ele deve)
  - Se há janela entre SIGTERM e remoção dos Endpoints (deve ser próxima de zero com readiness rápida)
- [ ] Comparar com um Deployment sem preStop e com readinessProbe lenta
- [ ] Deletar os recursos ao final

**Verificação:**
```bash
# Observar timestamps de eventos durante o rollout
kubectl get events --field-selector involvedObject.name=<pod-antigo> --watch

# Sequência esperada:
# 1. readinessProbe falha → Pod sai dos Endpoints
# 2. preStop hook executa (10s)
# 3. SIGTERM enviado
# 4. Container termina
# Não deve haver janela onde Pod recebe tráfego após SIGTERM
```

---

> [!tip] Ordem recomendada
> Exercícios 1 → 2 → 3 → 4 são sequenciais — cada um adiciona uma probe nova.
> Exercício 5 consolida tudo em uma configuração de produção completa — faça após 1-4.
> Exercício 6 é diagnóstico puro — pode ser feito a qualquer momento.
> Exercícios 7 e 8 são avançados e independentes — faça quando confortável com 1-6.
