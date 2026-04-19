---
tags:
  - exercicios
  - kubernetes
  - labels
  - annotations
  - selectors
tipo: exercicios
area: kubernetes
conteudo: "[[05-Kubernetes/labels-annotations]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Exercícios — Labels e Annotations no Kubernetes

> Conteúdo: [[05-Kubernetes/labels-annotations]] | Trilha: [[00-Trilha/kubernetes]]

**Infraestrutura:** k8s-master (192.168.3.30), worker1 (.31), worker2 (.32)
**Namespace de trabalho:** `estudo`
**Atenção:** Nunca tocar em `controle-gastos`, `promobot`, `databases/postgres`.

---

## Exercício 1: Inventário de labels no cluster (leitura apenas)

**Contexto:** Você assumiu a administração de um cluster herdado. Antes de padronizar os labels, precisa entender o que existe: quais labels são usados, se há consistência entre namespaces, e quais ferramentas estão configuradas via annotations.

**Missão:** Mapeie os labels e annotations dos recursos existentes no cluster sem alterar nada.

**Requisitos:**
- [ ] Listar todos os labels dos Pods no namespace `databases`: `kubectl get pods -n databases --show-labels`
- [ ] Listar todos os labels dos nodes: `kubectl get nodes --show-labels`
- [ ] Identificar quais labels automáticos o Kubernetes adicionou nos nodes (prefixo `kubernetes.io/` e `node.kubernetes.io/`)
- [ ] Ver os labels do namespace `databases`: `kubectl get namespace databases --show-labels` — o label `kubernetes.io/metadata.name` existe?
- [ ] Para o nginx Ingress Controller (namespace `ingress-nginx`): listar todas as annotations dos recursos e identificar quais ferramentas as colocaram
- [ ] Verificar as annotations dos Pods em `databases` — algum tem `prometheus.io/scrape`?
- [ ] Documentar: qual padrão de labels o cluster atual usa? Há consistência com `app.kubernetes.io/*`?

**Verificação:**
```bash
kubectl get pods -n databases --show-labels
# Deve mostrar labels dos Pods postgres

kubectl get nodes -o custom-columns='NAME:.metadata.name,LABELS:.metadata.labels'
# Todos os labels de cada node

kubectl get namespace databases -o jsonpath='{.metadata.labels}' | jq .
# Labels do namespace, incluindo kubernetes.io/metadata.name
```

---

## Exercício 2: Labels e selectors básicos

**Contexto:** O time está padronizando labels em novos Deployments. Você vai criar recursos com labels estruturados e demonstrar como os selectors funcionam na prática.

**Missão:** Crie múltiplos Pods com labels variados e pratique todos os tipos de selector.

**Requisitos:**
- [ ] Criar namespace `estudo`
- [ ] Criar 4 Pods com combinações diferentes de labels:
  - `pod-prod-v1`: `app=webapp, env=producao, version=1.0, tier=frontend`
  - `pod-prod-v2`: `app=webapp, env=producao, version=2.0, tier=frontend`
  - `pod-staging`: `app=webapp, env=staging, version=2.0, tier=frontend`
  - `pod-api`: `app=api, env=producao, version=1.0, tier=backend`
- [ ] Filtrar apenas os Pods de produção: `-l env=producao`
- [ ] Filtrar apenas os Pods do webapp em qualquer ambiente: `-l app=webapp`
- [ ] Filtrar Pods com versão 2.0 em qualquer app: `-l version=2.0`
- [ ] Usar set-based: Pods onde env está em (producao, staging): `-l 'env in (producao,staging)'`
- [ ] Usar set-based: Pods que NÃO têm a label `tier=backend`: `-l 'tier notin (backend)'`
- [ ] Usar exists: Pods que TÊM o label `version` (qualquer valor): `-l version`
- [ ] Usar doesnotexist: Pods que NÃO TÊM o label `debug`: `-l '!debug'`
- [ ] Mostrar colunas personalizadas com os labels: `-L app,env,version`

**Verificação:**
```bash
kubectl get pods -n estudo -l env=producao
# pod-prod-v1, pod-prod-v2, pod-api

kubectl get pods -n estudo -l 'env in (producao,staging)'
# pod-prod-v1, pod-prod-v2, pod-staging, pod-api

kubectl get pods -n estudo -L app,env,version
# Colunas APP, ENV, VERSION para cada Pod
```

---

## Exercício 3: Service com mismatch de labels — diagnóstico e correção

**Contexto:** Um Service foi criado mas a aplicação está inacessível. O desenvolvedor diz "os Pods estão Running". Você precisa diagnosticar e corrigir o problema sem reiniciar nada.

**Missão:** Reproduza o mismatch de labels, execute o diagnóstico completo e corrija sem recriar nenhum Pod.

**Requisitos:**
- [ ] Criar Deployment `minha-app` com 3 réplicas, label nos Pods: `app=minha-app-v2`
- [ ] Criar Service `minha-app-svc` com selector: `app=minha-app` (typo intencional — sem o `-v2`)
- [ ] Confirmar o sintoma: `kubectl get endpoints minha-app-svc -n estudo` → `<none>`
- [ ] Executar o diagnóstico completo:
  1. `kubectl describe svc minha-app-svc -n estudo` → ver o selector
  2. `kubectl get pods -n estudo --show-labels` → ver os labels reais
  3. `kubectl get pods -n estudo -l app=minha-app` → confirmar: zero resultados
  4. `kubectl get pods -n estudo -l app=minha-app-v2` → confirmar: 3 resultados
- [ ] Corrigir **sem deletar os Pods** — duas opções, escolha a mais apropriada:
  - Opção A: Editar o Service para usar o selector `app=minha-app-v2`
  - Opção B: Adicionar o label `app=minha-app` nos Pods existentes via `kubectl label`
- [ ] Confirmar que os Endpoints aparecem após a correção
- [ ] Documentar: qual opção escolheu e por quê?

**Verificação:**
```bash
kubectl get endpoints minha-app-svc -n estudo
# Antes: <none>
# Após correção: 3 IPs dos Pods

kubectl describe svc minha-app-svc -n estudo
# Selector correto e Endpoints populados
```

---

## Exercício 4: Selector imutável — demonstrar o problema e o workaround

**Contexto:** Um desenvolvedor quer mudar o selector de um Deployment para incluir um label de versão. Ele tenta `kubectl edit` e recebe um erro. Você precisa demonstrar a limitação e executar a migração correta.

**Missão:** Demonstre a imutabilidade do selector e execute a migração sem downtime.

**Requisitos:**
- [ ] Criar Deployment `app-imutavel` com selector `app=app-imutavel` e 2 réplicas
- [ ] Tentar mudar o selector via `kubectl edit deployment app-imutavel -n estudo` — adicionar `version: "1.0"` ao selector
- [ ] Registrar a mensagem de erro exata
- [ ] Executar a migração correta para adicionar `version: "1.0"` ao selector:
  1. Exportar o YAML atual: `kubectl get deployment app-imutavel -n estudo -o yaml > app-imutavel.yaml`
  2. Editar o arquivo: mudar `spec.selector.matchLabels` e `spec.template.metadata.labels`
  3. Deletar o Deployment antigo (os Pods continuam rodando temporariamente pois o ReplicaSet ainda existe)
  4. Aplicar o novo YAML
- [ ] Verificar que os Pods antigos (com labels antigos) ficaram órfãos e precisam ser deletados manualmente
- [ ] Deletar os Pods órfãos identificando-os pelo label antigo
- [ ] Confirmar o estado final: apenas Pods com o novo selector rodando

**Verificação:**
```bash
kubectl edit deployment app-imutavel -n estudo
# Error: spec.selector: Invalid value: ... field is immutable

kubectl get pods -n estudo --show-labels
# Após a migração: Pods antigos (app=app-imutavel sem version) + Pods novos (app=app-imutavel, version=1.0)
# Os Pods antigos estão órfãos — o novo ReplicaSet não os gerencia

kubectl get pods -n estudo -l app=app-imutavel,version=1.0
# Apenas os Pods gerenciados pelo novo Deployment
```

---

## Exercício 5: Annotations para ferramentas — change-cause e Prometheus

**Contexto:** O time quer dois comportamentos: (1) o histórico de rollout deve ter mensagens descritivas de cada mudança; (2) o Prometheus deve fazer scraping automático de uma nova aplicação via annotations nos Pods.

**Missão:** Configure o `change-cause` para enriquecer o histórico de rollouts e adicione annotations de scraping do Prometheus nos Pods.

**Requisitos:**
- [ ] Criar Deployment `monitored-app` com imagem `nginx:1.26` no namespace `estudo`
- [ ] Anotar o deploy inicial com change-cause:
  ```bash
  kubectl annotate deployment monitored-app -n estudo \
    kubernetes.io/change-cause="Deploy inicial — nginx 1.26"
  ```
- [ ] Verificar o histórico: `kubectl rollout history deployment/monitored-app -n estudo`
- [ ] Atualizar a imagem para `nginx:1.27` e anotar antes ou depois:
  ```bash
  kubectl set image deployment/monitored-app app=nginx:1.27 -n estudo
  kubectl annotate deployment monitored-app -n estudo \
    kubernetes.io/change-cause="Atualização de segurança — nginx 1.27" --overwrite
  ```
- [ ] Confirmar que o histórico agora mostra as mensagens descritivas
- [ ] Adicionar annotations de Prometheus **no template dos Pods** do Deployment (não no Deployment em si):
  ```yaml
  spec:
    template:
      metadata:
        annotations:
          prometheus.io/scrape: "true"
          prometheus.io/port: "80"
          prometheus.io/path: "/metrics"
  ```
- [ ] Confirmar que os Pods têm as annotations: `kubectl get pod <nome> -n estudo -o jsonpath='{.metadata.annotations}'`
- [ ] Documentar: por que as annotations de Prometheus ficam no template dos Pods e não no Deployment?

**Verificação:**
```bash
kubectl rollout history deployment/monitored-app -n estudo
# REVISION  CHANGE-CAUSE
# 1         Deploy inicial — nginx 1.26
# 2         Atualização de segurança — nginx 1.27

kubectl get pods -n estudo -l app=monitored-app \
  -o jsonpath='{range .items[*]}{.metadata.name}: {.metadata.annotations.prometheus\.io/scrape}{"\n"}{end}'
# Deve mostrar "true" para cada Pod
```

---

## Exercício 6: Labels em nodes e nodeSelector

**Contexto:** O time de dados precisa que certos Pods de processamento rodem apenas em `k8s-worker2` (que tem mais memória). Você vai usar labels em nodes e nodeSelector para garantir esse placement.

**Missão:** Adicione um label ao node correto e use nodeSelector para direcionar o scheduling.

**Requisitos:**
- [ ] Verificar os labels atuais dos dois workers: `kubectl get nodes --show-labels`
- [ ] Adicionar label `workload=data-processing` ao `k8s-worker2`:
  ```bash
  kubectl label node k8s-worker2 workload=data-processing
  ```
- [ ] Criar Deployment `data-processor` com `nodeSelector: {workload: data-processing}` e 3 réplicas
- [ ] Verificar em quais nodes os Pods foram agendados: `kubectl get pods -n estudo -o wide -l app=data-processor`
- [ ] Confirmar que nenhum Pod foi para `k8s-worker1`
- [ ] **Cenário de falha:** remover o label do node:
  ```bash
  kubectl label node k8s-worker2 workload-
  ```
- [ ] Escalar o Deployment para 4 réplicas — o que acontece com o Pod novo?
- [ ] Documentar: quais Pods ficam `Pending`? Por quê? Como resolver?
- [ ] Restaurar o label no node e confirmar que o Pod Pending é agendado

**Verificação:**
```bash
kubectl get pods -n estudo -o wide -l app=data-processor
# NODE: k8s-worker2 para todos os Pods

kubectl label node k8s-worker2 workload-
kubectl scale deployment data-processor --replicas=4 -n estudo
kubectl get pods -n estudo -l app=data-processor
# Um Pod em Pending (sem node com o label)

kubectl describe pod <pod-pending> -n estudo
# Events: 0/2 nodes are available: 2 node(s) didn't match node selector
```

---

## Exercício 7: jsonpath e custom-columns para diagnóstico avançado

**Contexto:** O time de operações quer um one-liner que mostre, para cada Pod do cluster: nome, namespace, node, app label, e status — em formato tabular. Você vai construir essa query usando `custom-columns` e `jsonpath`.

**Missão:** Construa queries avançadas com labels e annotations via jsonpath e custom-columns.

**Requisitos:**
- [ ] Listar todos os Pods do namespace `estudo` com colunas: `NAME`, `APP`, `ENV`, `VERSION`, `NODE`, `STATUS`:
  ```bash
  kubectl get pods -n estudo -o custom-columns=\
  'NAME:.metadata.name,APP:.metadata.labels.app,ENV:.metadata.labels.env,NODE:.spec.nodeName,STATUS:.status.phase'
  ```
- [ ] Usar jsonpath para listar apenas o nome e a annotation `change-cause` do Deployment:
  ```bash
  kubectl get deployments -n estudo \
    -o jsonpath='{range .items[*]}{.metadata.name}: {.metadata.annotations.kubernetes\.io/change-cause}{"\n"}{end}'
  ```
- [ ] Listar todos os Pods do cluster que têm a annotation `prometheus.io/scrape=true` (usando `jq`):
  ```bash
  kubectl get pods -A -o json | \
    jq '.items[] | select(.metadata.annotations["prometheus.io/scrape"] == "true") | "\(.metadata.namespace)/\(.metadata.name)"'
  ```
- [ ] Listar nodes ordenados pela quantidade de Pods agendados neles:
  ```bash
  kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort | uniq -c | sort -rn
  ```
- [ ] Encontrar todos os Pods sem o label `app` (Pods sem label de identificação):
  ```bash
  kubectl get pods -A -o json | jq '.items[] | select(.metadata.labels.app == null) | "\(.metadata.namespace)/\(.metadata.name)"'
  ```

**Verificação:**
```bash
# As queries devem retornar resultados sem erros de sintaxe
# custom-columns: tabela formatada
# jsonpath: uma linha por objeto
# jq: lista de namespaces/pods

kubectl get pods -n estudo -o custom-columns='NAME:.metadata.name,APP:.metadata.labels.app'
# Tabela com nome e app label de cada Pod
```

---

## Exercício 8: Label hijack — ReplicaSet adotando Pod manual (Staff-level)

**Contexto:** Um SRE júnior criou um Pod de debug manualmente no namespace `estudo` com os mesmos labels do Deployment de produção. Alguns minutos depois, ele percebeu que um Pod do Deployment foi deletado "sozinho". Você precisa reproduzir e explicar o fenômeno.

**Missão:** Reproduza o label hijack, observe o comportamento do ReplicaSet, e implemente a prática correta para evitar isso.

**Requisitos:**
- [ ] Criar Deployment `webapp-hs` com 3 réplicas e selector/labels `app=webapp-hs`
- [ ] Confirmar que os 3 Pods estão Running
- [ ] Criar Pod manual com os **mesmos labels** que o ReplicaSet gerencia:
  ```bash
  kubectl run pod-debug -n estudo \
    --image=nginx:1.27 \
    --labels="app=webapp-hs" \
    --restart=Never
  ```
- [ ] Observar imediatamente o que acontece com os Pods: `kubectl get pods -n estudo -w`
- [ ] O ReplicaSet "adotou" o Pod manual? O número total de Pods ultrapassou 3?
- [ ] Um Pod gerenciado foi terminado para manter `replicas: 3`?
- [ ] Verificar o campo `ownerReferences` do Pod manual adotado:
  ```bash
  kubectl get pod pod-debug -n estudo -o jsonpath='{.metadata.ownerReferences}'
  ```
- [ ] **Correção:** Quais labels usar em Pods de debug para evitar adoção? Criar um novo Pod de debug com labels que não conflitem com nenhum selector existente:
  - Sugestão: `role=debug,debug-session=2025-04-18`
- [ ] Confirmar que o Pod com labels de debug não é adotado por nenhum ReplicaSet
- [ ] Limpeza final: `kubectl delete namespace estudo`

**Verificação:**
```bash
kubectl get pods -n estudo -w
# Após criar pod-debug com app=webapp-hs:
# Um Pod gerenciado vai para Terminating imediatamente (RS mantém 3)

kubectl get pod pod-debug -n estudo -o jsonpath='{.metadata.ownerReferences[0].kind}'
# ReplicaSet (foi adotado!)

kubectl get pods -n estudo
# Total: 3 Pods (2 gerenciados originais + pod-debug adotado = 3, um original foi deletado)
```

---

> [!tip] Ordem recomendada
> Exercício 1 é leitura obrigatória — entender o que existe antes de criar.
> Exercícios 2 e 3 são os mais importantes para o dia a dia — selectors e mismatch de labels.
> Exercício 4 demonstra uma limitação crítica do Kubernetes que pega todos de surpresa.
> Exercício 5 é independente — pode ser feito em qualquer ordem após o 2.
> Exercício 6 requer acesso ao node para adicionar labels — fácil mas importante para entender scheduling.
> Exercício 7 é técnico mas muito útil no dia a dia de operações — queries avançadas.
> Exercício 8 é o mais surpreendente — reserve para quando estiver confiante com os anteriores.

> [!warning] Limpeza
> `kubectl delete namespace estudo` remove todos os recursos criados.
> Labels adicionados a **nodes** não são removidos com o namespace — lembrar de remover:
> `kubectl label node k8s-worker2 workload-`

> [!info] Labels em nodes são persistentes
> Labels em nodes sobrevivem a reinicializações do cluster e a deploys de novos objetos. Ao final dos exercícios, verificar se o label `workload=data-processing` ainda está no worker2 e removê-lo se necessário.
