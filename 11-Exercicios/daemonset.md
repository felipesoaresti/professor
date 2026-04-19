---
tags:
  - exercicios
  - kubernetes
  - daemonset
  - workloads
tipo: exercicios
area: kubernetes
conteudo: "[[05-Kubernetes/daemonset]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Exercícios — DaemonSet

> Conteúdo: [[05-Kubernetes/daemonset]] | Trilha: [[00-Trilha/kubernetes]]

**Infraestrutura:** k8s-master (192.168.3.30), worker1 (192.168.3.31), worker2 (192.168.3.32)
**Namespace de trabalho:** `default` ou criar `monitoring` para estes exercícios
**Atenção:** Não tocar em `controle-gastos`, `promobot`, `databases/postgres`.

---

## Exercício 1: Primeiro DaemonSet — entendendo a distribuição por node

**Contexto:** O time de plataforma precisa instalar um agente simples de coleta de métricas em todos os workers. Você vai criar o DaemonSet e confirmar que ele distribui corretamente.

**Missão:** Crie um DaemonSet básico e observe como ele se comporta em relação aos nodes do cluster.

**Requisitos:**
- [ ] Criar namespace `monitoring`
- [ ] Criar DaemonSet `node-agent` no namespace `monitoring` usando `busybox:1.36`
- [ ] O container deve executar um loop infinito que imprime o nome do node a cada 30s (usar Downward API para injetar `spec.nodeName`)
- [ ] **Não adicionar** toleration para o control-plane inicialmente
- [ ] Confirmar quantos Pods foram criados e em quais nodes
- [ ] Verificar: o master (k8s-master) tem um Pod? Por quê?
- [ ] Registrar os campos `DESIRED`, `CURRENT`, `READY` do `kubectl get ds`
- [ ] Confirmar via `kubectl logs` que o container está imprimindo o nome do node correto

**Verificação:**
```bash
kubectl get ds -n monitoring
# DESIRED deve ser 2 (apenas workers, sem o master)

kubectl get pods -n monitoring -o wide
# Pods devem estar em worker1 e worker2, NÃO no master

kubectl logs -n monitoring -l app=node-agent
# Deve mostrar o nome do node em cada linha
```

---

## Exercício 2: Incluindo o control-plane — tolerations

**Contexto:** O time de segurança quer o agente em **todos** os nodes, incluindo o master. O node master tem um taint que impede Pods normais.

**Missão:** Identifique o taint do control-plane e configure a toleration correta para que o DaemonSet rode em todos os 3 nodes.

**Requisitos:**
- [ ] Verificar os taints do k8s-master via `kubectl describe node k8s-master`
- [ ] Anotar o nome exato do taint e o effect
- [ ] Adicionar a toleration correta ao DaemonSet `node-agent` (usar `kubectl edit` ou `kubectl patch`)
- [ ] Confirmar que o `DESIRED` mudou de 2 para 3
- [ ] Confirmar que agora existe um Pod no k8s-master
- [ ] Verificar os logs do Pod no master para confirmar que ele imprime o nome correto do node

**Verificação:**
```bash
kubectl get ds -n monitoring
# DESIRED: 3  CURRENT: 3  READY: 3

kubectl get pods -n monitoring -o wide
# Deve haver um Pod em cada um dos 3 nodes: k8s-master, k8s-worker1, k8s-worker2
```

---

## Exercício 3: DaemonSet em subset de nodes com nodeSelector

**Contexto:** Um novo tipo de workload requer um agente especial apenas nos nodes de storage. Você vai usar labels de node para restringir o DaemonSet a um subset.

**Missão:** Crie um DaemonSet que rode apenas em nodes com um label específico, e teste o comportamento quando o label é adicionado/removido de um node.

**Requisitos:**
- [ ] Criar DaemonSet `storage-agent` no namespace `monitoring` com `nodeSelector: {role: storage}` usando `busybox:1.36`
- [ ] Verificar que nenhum Pod foi criado (nenhum node tem o label ainda)
- [ ] Adicionar label `role=storage` ao k8s-worker1: `kubectl label node k8s-worker1 role=storage`
- [ ] Confirmar que o DaemonSet criou exatamente 1 Pod, apenas no worker1
- [ ] Adicionar o mesmo label ao k8s-worker2
- [ ] Confirmar que agora há 2 Pods (worker1 e worker2)
- [ ] Remover o label do worker1: `kubectl label node k8s-worker1 role-`
- [ ] Confirmar que o Pod do worker1 foi **deletado automaticamente**
- [ ] Limpar: remover o label do worker2 e deletar o DaemonSet

**Verificação:**
```bash
kubectl get pods -n monitoring -l app=storage-agent -o wide -w
# Observar em tempo real: pod criado quando label adicionado, deletado quando removido
```

---

## Exercício 4: Inspecionando o Calico DaemonSet (DaemonSet real de produção)

**Contexto:** O Calico CNI já está rodando como DaemonSet no cluster. Você vai analisar como um DaemonSet de produção real é configurado.

**Missão:** Inspecione o DaemonSet `calico-node` e documente as configurações que o tornam apto a rodar como plugin de rede crítico.

**Requisitos:**
- [ ] Listar todos os DaemonSets do namespace `kube-system`
- [ ] Fazer `kubectl describe ds calico-node -n kube-system` e registrar:
  - Quantos nodes têm o Pod?
  - Quais tolerations estão configuradas?
  - O DaemonSet usa `hostNetwork`?
  - Quais `hostPath` volumes estão montados e por quê?
  - Quais variáveis de ambiente usam Downward API?
- [ ] Ver o YAML completo: `kubectl get ds calico-node -n kube-system -o yaml`
- [ ] Identificar qual `updateStrategy` está configurada
- [ ] Comparar com o DaemonSet `node-agent` que você criou: quais diferenças justificam a complexidade do Calico?

**Verificação:**
```bash
kubectl describe ds calico-node -n kube-system | grep -E "Tolerations|Host Network|Volumes|Update Strategy"
# Deve mostrar tolerations para control-plane, hostNetwork: true
```

---

## Exercício 5: RollingUpdate do DaemonSet

**Contexto:** Uma nova versão do agente de monitoramento foi lançada. Você precisa atualizar o DaemonSet com zero-downtime, controlando o ritmo de atualização (um node por vez).

**Missão:** Atualize o DaemonSet `node-agent` de `busybox:1.36` para `busybox:1.37` usando RollingUpdate com `maxUnavailable: 1` e monitore o processo.

**Requisitos:**
- [ ] Confirmar que o DaemonSet `node-agent` está rodando com `busybox:1.36` nos 3 nodes
- [ ] Verificar a `updateStrategy` atual (deve ser RollingUpdate)
- [ ] Atualizar a imagem: `kubectl set image ds/node-agent agent=busybox:1.37 -n monitoring`
- [ ] Monitorar o rollout com `kubectl rollout status daemonset/node-agent -n monitoring`
- [ ] Em paralelo, observar `kubectl get pods -n monitoring -w` para ver a atualização node a node
- [ ] Verificar o histórico de revisões: `kubectl rollout history daemonset/node-agent -n monitoring`
- [ ] Fazer rollback para a versão anterior e confirmar que voltou ao `busybox:1.36`

**Verificação:**
```bash
kubectl rollout status daemonset/node-agent -n monitoring
# daemon set "node-agent" successfully rolled out

kubectl get pods -n monitoring -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'
# Todos devem mostrar busybox:1.37 (ou busybox:1.36 após rollback)
```

---

## Exercício 6: DaemonSet com hostPath — lendo arquivos do node

**Contexto:** O novo agente de log precisa ler os logs do sistema em `/var/log/syslog` de cada node para enviá-los ao servidor de logs centralizado.

**Missão:** Configure um DaemonSet que monta o diretório `/var/log` do host como volume e lê logs do sistema.

**Requisitos:**
- [ ] Criar DaemonSet `log-collector` no namespace `monitoring`
- [ ] Configurar `hostPath` volume apontando para `/var/log` do node
- [ ] Montar o volume no container em `/host/var/log` com `readOnly: true`
- [ ] O container deve executar: `tail -f /host/var/log/syslog` (ou `/host/var/log/kern.log` se syslog não existir)
- [ ] Confirmar via `kubectl logs` que o container está lendo os logs do node host
- [ ] Verificar que o conteúdo visto dentro do Pod é o mesmo que em `/var/log/syslog` no node (acessar via SSH e comparar)
- [ ] Deletar o DaemonSet ao final

**Verificação:**
```bash
kubectl logs -n monitoring -l app=log-collector --tail=10
# Deve mostrar linhas do syslog do node host

# SSH no worker e comparar:
tail -5 /var/log/syslog   # (ou kern.log)
# As linhas devem ser idênticas ao kubectl logs
```

---

## Exercício 7: DaemonSet vs Deployment — prova empírica (avançado)

**Contexto:** Um colega argumenta que "basta usar um Deployment com antiAffinity para garantir um Pod por node". Você vai provar empiricamente por que isso não é equivalente a um DaemonSet.

**Missão:** Crie um Deployment com `podAntiAffinity` requiredDuringScheduling e compare o comportamento com um DaemonSet ao adicionar um novo node simulado (usando label para restringir e depois expandir).

**Requisitos:**
- [ ] Criar Deployment `agent-deploy` com 2 réplicas e `podAntiAffinity` que impede 2 Pods no mesmo node
- [ ] Criar DaemonSet `agent-ds` com o mesmo workload
- [ ] Confirmar que ambos têm 1 Pod em cada worker (worker1 e worker2)
- [ ] Simular "adicionar um node" adicionando label ao master: adicionar toleration ao DaemonSet para o master e observar que o DS cria um 3º Pod automaticamente
- [ ] Verificar que o Deployment ainda tem apenas 2 réplicas — não criou Pod no master automaticamente
- [ ] Documentar: qual seria o procedimento manual para o Deployment cobrir o novo node? Por que é inferior?
- [ ] Limpar todos os recursos ao final

**Verificação:**
```bash
kubectl get pods -l app=agent-deploy -o wide
# 2 pods, apenas nos 2 workers — não expandiu para o master

kubectl get pods -l app=agent-ds -o wide
# 3 pods — master, worker1, worker2 (após adicionar toleration)
```

---

## Exercício 8: DaemonSet com probes e recursos — configuração de produção (Staff-level)

**Contexto:** O DaemonSet `node-agent` vai para produção. O SRE exige: probes configuradas, resources definidos, graceful shutdown, e priorityClass adequada para sobreviver a pressão de memória.

**Missão:** Refaça o DaemonSet `node-agent` com configuração completa de produção.

**Requisitos:**
- [ ] Configurar `resources.requests: {cpu: 50m, memory: 32Mi}` e `resources.limits: {cpu: 100m, memory: 64Mi}`
- [ ] Configurar `livenessProbe` exec que verifica se o processo está rodando (`pgrep sh` ou similar), `periodSeconds: 30`, `failureThreshold: 3`
- [ ] Configurar `readinessProbe` exec que verifica se o agente está coletando (`test -f /tmp/healthy`), `periodSeconds: 10`
- [ ] Adicionar `terminationGracePeriodSeconds: 30`
- [ ] Configurar `updateStrategy.rollingUpdate.maxUnavailable: 1` e `minReadySeconds: 15`
- [ ] Toleration para control-plane (roda em todos os nodes)
- [ ] Injetar `NODE_NAME` e `NODE_IP` via Downward API
- [ ] Confirmar que todos os 3 Pods ficam `1/1 Ready`
- [ ] Testar a livenessProbe: matar o processo dentro do container e observar o restart

**Verificação:**
```bash
kubectl get ds node-agent -n monitoring
# DESIRED: 3  CURRENT: 3  READY: 3

kubectl describe ds node-agent -n monitoring | grep -E "Liveness|Readiness|Requests|Limits|Tolerations"
# Todos os campos devem estar presentes

kubectl get pods -n monitoring -o wide
# Todos em Running 1/1, distribuídos nos 3 nodes
```

---

> [!tip] Ordem recomendada
> Exercícios 1 → 2 → 3 são sequenciais — entendem a distribuição por node progressivamente.
> Exercício 4 é análise pura de produção — faça após entender o básico (ex. 1-3).
> Exercício 5 testa update — faça após exercício 2 (precisa do DaemonSet com 3 nodes).
> Exercícios 6, 7, 8 são independentes — avançados, qualquer ordem após 1-5.

> [!warning] Limpar recursos
> Ao final dos exercícios, deletar o namespace `monitoring` para liberar recursos:
> `kubectl delete namespace monitoring`
