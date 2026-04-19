---
tags:
  - kubernetes
  - labels
  - annotations
  - selectors
  - metadados
area: kubernetes
tipo: conteudo
prerequisites:
  - "[[05-Kubernetes/kubernetes-teoria-inicial]]"
next:
  - "[[11-Exercicios/labels-annotations]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Labels e Annotations no Kubernetes

> Pré-requisito: [[05-Kubernetes/kubernetes-teoria-inicial]] | Exercícios: [[11-Exercicios/labels-annotations]] | Trilha: [[00-Trilha/kubernetes]]

---

## O que são e por que existem

Todo objeto Kubernetes tem um campo `metadata`. Dentro dele, dois subcampos carregam metadados em forma de pares chave/valor:

- **Labels** — para **identificação e seleção**. Controllers, Services e NetworkPolicies usam labels para encontrar os objetos que gerenciam.
- **Annotations** — para **metadados arbitrários**. Ferramentas externas (cert-manager, Prometheus, nginx) usam annotations para configuração. Não são usadas para seleção.

A distinção fundamental:

| | Labels | Annotations |
|---|---|---|
| Usados para seleção? | **Sim** — por Services, ReplicaSets, NetworkPolicies | **Não** |
| Tamanho da chave/valor | Limitado (63 chars por parte) | Ilimitado (valores grandes são ok) |
| Indexados no etcd | **Sim** — busca eficiente com `-l` | Não |
| Quem os usa | Controllers do Kubernetes | Ferramentas externas, admission webhooks |
| Exemplo de uso | Service encontra Pods | cert-manager configura emissão de cert |

---

## Como funcionam internamente

### Formato de chave

Ambos usam o mesmo formato de chave:

```
[prefix/]name
```

- **prefix** (opcional): domínio DNS reverso. Máximo 253 chars. Separado por `/`.
- **name**: máximo 63 chars. Apenas `[a-z0-9A-Z]`, `-`, `_`, `.`. Deve começar e terminar com alfanumérico.

Exemplos válidos:
```
app
version
app.kubernetes.io/name
nginx.ingress.kubernetes.io/rewrite-target
cert-manager.io/cluster-issuer
prometheus.io/scrape
```

Prefixos reservados:
- `kubernetes.io/` e `k8s.io/` — uso exclusivo do Kubernetes core
- Qualquer outra ferramenta deve usar seu próprio domínio como prefixo

### Como labels são indexados no etcd

O kube-apiserver mantém um índice de labels no etcd. Quando você faz `kubectl get pods -l app=nginx`, o apiserver faz uma busca eficiente por esse índice — não varre todos os objetos. Por isso labels são adequados para seleção de alta frequência (Services, ReplicaSets, NetworkPolicies precisam selecionar Pods continuamente).

Annotations não são indexadas — uma busca por annotation precisaria varrer todos os objetos.

### Label selectors — como os controllers encontram objetos

Existem dois tipos de selector:

**Equality-based** (simples):
```yaml
selector:
  app: nginx
  env: producao
```
Combina exatamente: `app=nginx AND env=producao`.

**Set-based** (mais expressivo, usado em Deployments, Jobs, NetworkPolicies):
```yaml
selector:
  matchLabels:
    app: nginx
  matchExpressions:
  - key: env
    operator: In
    values: [producao, staging]
  - key: debug
    operator: DoesNotExist
```

Operadores disponíveis: `In`, `NotIn`, `Exists`, `DoesNotExist`.

### Por que o selector do Deployment é imutável

Uma vez criado, o `spec.selector` de um Deployment, ReplicaSet, StatefulSet e DaemonSet é **imutável**. Motivo: se o selector mudasse, os Pods antigos deixariam de ser gerenciados pelo controller mas continuariam rodando — orçãos. O Kubernetes força você a deletar e recriar o recurso com novo selector para evitar esse estado.

```bash
kubectl edit deployment webapp
# Tentar mudar spec.selector → erro:
# The Deployment "webapp" is invalid: spec.selector: Invalid value: ...
# field is immutable
```

---

## Na prática — comandos e exemplos reais

### Adicionar labels e annotations na criação (YAML)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: estudo
  labels:
    app.kubernetes.io/name: webapp
    app.kubernetes.io/version: "2.1.0"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: ecommerce
    env: producao
  annotations:
    deployment.kubernetes.io/revision: "1"
    description: "Servidor web principal do ecommerce"
    contact: "felipe@staypuff.info"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp          # ← selector imutável — use chaves simples aqui
  template:
    metadata:
      labels:
        app: webapp
        app.kubernetes.io/name: webapp
        version: "2.1.0"
        env: producao
    spec:
      containers:
      - name: app
        image: nginx:1.27
```

### Adicionar/remover labels e annotations imperativamente

```bash
# Adicionar label a um Pod
kubectl label pod meu-pod -n estudo version=1.2

# Sobrescrever label existente (requer --overwrite)
kubectl label pod meu-pod -n estudo version=1.3 --overwrite

# Remover label (sufixo -)
kubectl label pod meu-pod -n estudo version-

# Adicionar annotation
kubectl annotate pod meu-pod -n estudo \
  description="Pod de teste para validação de deploy"

# Remover annotation
kubectl annotate pod meu-pod -n estudo description-

# Label em múltiplos objetos de uma vez
kubectl label pods -l app=webapp -n estudo env=staging --overwrite

# Label em um node
kubectl label node k8s-worker1 disk=ssd
```

### Filtrar com selectors

```bash
# Equality-based
kubectl get pods -n estudo -l app=webapp
kubectl get pods -n estudo -l app=webapp,env=producao

# Set-based (usar aspas)
kubectl get pods -n estudo -l 'env in (producao,staging)'
kubectl get pods -n estudo -l 'env notin (debug)'
kubectl get pods -n estudo -l 'version'           # apenas Pods que TÊM a label version
kubectl get pods -n estudo -l '!debug'            # apenas Pods que NÃO TÊM a label debug

# Mostrar os labels no output
kubectl get pods -n estudo --show-labels
kubectl get pods -n estudo -L app,env             # colunas por label específico

# Field selector (diferente de label selector — filtra por campos do objeto)
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector spec.nodeName=k8s-worker1
kubectl get pods --field-selector metadata.namespace=estudo

# Combinar label selector e field selector
kubectl get pods -n estudo -l app=webapp \
  --field-selector status.phase=Running
```

### Inspecionar labels e annotations de um objeto

```bash
# Ver todos os labels e annotations
kubectl get pod meu-pod -n estudo -o yaml | grep -A10 "labels:"
kubectl get pod meu-pod -n estudo -o yaml | grep -A10 "annotations:"

# Via jsonpath
kubectl get pods -n estudo -o jsonpath='{range .items[*]}{.metadata.name}: {.metadata.labels}{"\n"}{end}'

# Ver apenas uma annotation específica
kubectl get ingress webapp -n estudo \
  -o jsonpath='{.metadata.annotations.nginx\.ingress\.kubernetes\.io/ssl-redirect}'
```

---

## Labels recomendados pelo Kubernetes

O Kubernetes define um conjunto de labels `app.kubernetes.io/*` que todos os objetos deveriam ter. Ferramentas como Helm, kustomize e dashboards os leem para organizar e exibir recursos:

| Label | Descrição | Exemplo |
|---|---|---|
| `app.kubernetes.io/name` | Nome da aplicação | `webapp` |
| `app.kubernetes.io/version` | Versão atual | `"2.1.0"` |
| `app.kubernetes.io/component` | Componente dentro da app | `frontend`, `backend`, `database` |
| `app.kubernetes.io/part-of` | Nome do sistema maior | `ecommerce` |
| `app.kubernetes.io/managed-by` | Ferramenta que gerencia | `helm`, `argocd`, `kubectl` |
| `app.kubernetes.io/instance` | Instância única da app | `webapp-producao` |

> [!tip] Helm usa esses labels automaticamente
> Quando você instala um chart Helm, ele aplica `app.kubernetes.io/managed-by: Helm` e `helm.sh/chart: <nome>-<versão>` em todos os recursos. Por isso `kubectl get all -n <ns> -l app.kubernetes.io/managed-by=Helm` lista tudo que o Helm gerencia.

---

## Annotations mais usadas por ferramentas

### cert-manager

```yaml
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-production"
  cert-manager.io/issuer: "letsencrypt-staging"        # namespace-scoped
  cert-manager.io/common-name: "estudo.staypuff.info"
```

### nginx Ingress Controller

```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
  nginx.ingress.kubernetes.io/rewrite-target: /$2
  nginx.ingress.kubernetes.io/proxy-body-size: "50m"
  nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
  nginx.ingress.kubernetes.io/limit-rps: "10"
```

### Prometheus (scraping automático)

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
```

O Prometheus Operator lê essas annotations nos Pods e configura scraping automaticamente — sem precisar criar `ServiceMonitor` manualmente.

### kubectl.kubernetes.io (annotations do sistema)

```yaml
annotations:
  kubectl.kubernetes.io/last-applied-configuration: "..."   # adicionado pelo kubectl apply
  deployment.kubernetes.io/revision: "3"                    # revisão atual do Deployment
  kubernetes.io/change-cause: "Update para nginx 1.27"      # histórico de rollout
```

A annotation `kubernetes.io/change-cause` aparece no `kubectl rollout history`:

```bash
kubectl annotate deployment webapp -n estudo \
  kubernetes.io/change-cause="Update nginx 1.26 → 1.27"

kubectl rollout history deployment/webapp -n estudo
# REVISION  CHANGE-CAUSE
# 1         Deploy inicial
# 2         Update nginx 1.26 → 1.27
```

### Anotações de scheduling e operação

```yaml
annotations:
  cluster-autoscaler.kubernetes.io/safe-to-evict: "false"    # não evictar este Pod
  karpenter.sh/do-not-evict: "true"                          # mesma função no Karpenter
  fluxcd.io/ignore: "true"                                   # GitOps: ignorar este recurso
```

---

## Casos de uso e boas práticas

### Separar selector de labels de metadados

O selector do Deployment (e ReplicaSet) deve ter **apenas o mínimo necessário** — porque é imutável. Labels de metadados ricos ficam no `metadata.labels` do Deployment e no template dos Pods, mas não precisam estar no selector:

```yaml
# Bom — selector mínimo, labels ricos no metadata
metadata:
  labels:
    app.kubernetes.io/name: webapp
    app.kubernetes.io/version: "2.1.0"
    env: producao
spec:
  selector:
    matchLabels:
      app: webapp          # ← apenas o necessário para identificar os Pods
  template:
    metadata:
      labels:
        app: webapp        # ← precisa bater com o selector
        version: "2.1.0"  # ← labels extras nos Pods (não precisam estar no selector)
        env: producao
```

### Nunca incluir versão no selector

`version` no selector cria um problema: quando você atualiza a versão, o selector muda — imutável. A versão deve ser label de metadados, não de seleção.

### Labels para Network Policy

NetworkPolicy usa labels para definir quais Pods podem se comunicar:

```yaml
spec:
  podSelector:
    matchLabels:
      app: webapp          # quais Pods esta policy afeta
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api         # apenas Pods com app=api podem entrar
```

### Annotations para configuração de ferramentas externas

Qualquer comportamento que não seja "encontrar este objeto" deve ser annotation, não label:
- Configuração do Ingress Controller → annotation
- Prometheus scraping → annotation
- Notas para operadores humanos → annotation
- Histórico de deploy → annotation

### Labels em nodes para scheduling

Labels em nodes permitem que Pods sejam agendados seletivamente:

```bash
# Adicionar label a um node
kubectl label node k8s-worker1 disk=ssd tier=storage

# Pod com nodeSelector
spec:
  nodeSelector:
    disk: ssd
```

Ver [[05-Kubernetes/daemonset]] para tolerations em nodes, e [[05-Kubernetes/statefulset-headless]] para uso de nodeAffinity.

---

## Troubleshooting — cenários reais de produção

### Service não enxerga os Pods (Endpoints vazios)

A causa mais comum: mismatch entre o selector do Service e os labels dos Pods.

```bash
# Ver o selector do Service
kubectl describe svc webapp-svc -n estudo | grep Selector
# Selector: app=webapp

# Ver os labels dos Pods
kubectl get pods -n estudo --show-labels
# NAME        LABELS
# webapp-abc  app=webapp-frontend,env=producao   ← "webapp-frontend", não "webapp"

# Confirmar: quantos Pods o selector encontra?
kubectl get pods -n estudo -l app=webapp
# No resources found  → mismatch confirmado

kubectl get endpoints webapp-svc -n estudo
# NAME         ENDPOINTS   AGE
# webapp-svc   <none>      5m
```

**Fix:** corrigir o label no Pod (editar o Deployment) ou corrigir o selector no Service (recriar, pois selector de Service não é imutável).

### ReplicaSet adotando Pods não gerenciados (label hijack)

Se você criar um Pod manualmente com os mesmos labels que um ReplicaSet espera, o ReplicaSet pode "adotar" esse Pod e considerá-lo como uma das suas réplicas — possivelmente deletando outros Pods para manter o número de réplicas:

```bash
# ReplicaSet com selector app=webapp, réplicas=3
# Você cria um Pod manual com label app=webapp

kubectl get pods -n estudo
# webapp-abc  Running  (gerenciado pelo RS)
# webapp-def  Running  (gerenciado pelo RS)
# webapp-ghi  Running  (gerenciado pelo RS)
# meu-pod     Running  (manual, mas com app=webapp ← adotado pelo RS!)
# webapp-jkl  Terminating  (RS deleta um para manter 3)
```

**Evitar:** use labels únicos em Pods manuais de debug.

### Annotation longa causando falha no apply

Annotations não têm limite de tamanho por design, mas o etcd tem limite de 1.5MB por objeto. Annotations com conteúdo muito longo (como `kubectl.kubernetes.io/last-applied-configuration` de YAMLs gigantes) podem causar:

```
The request is too large in bytes. Try a smaller request.
```

**Fix:** usar `kubectl create` em vez de `kubectl apply` para evitar a annotation de last-applied-configuration, ou usar Server-Side Apply (`kubectl apply --server-side`).

### Selector imutável — como atualizar

```bash
kubectl edit deployment webapp -n estudo
# erro: spec.selector: field is immutable

# Solução: recriar o Deployment
kubectl get deployment webapp -n estudo -o yaml > webapp.yaml
# Editar webapp.yaml: mudar selector e labels dos Pods
kubectl delete deployment webapp -n estudo
kubectl apply -f webapp.yaml
# Atenção: os Pods antigos ficam órfãos — deletar manualmente
kubectl delete pods -n estudo -l app=webapp-antigo
```

---

## Nível avançado — edge cases e cenários CKA/Staff

### jsonpath com labels para output customizado

```bash
# Nome + labels específicos de todos os Pods
kubectl get pods -n estudo \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.app}{"\t"}{.metadata.labels.version}{"\n"}{end}'

# custom-columns (mais legível)
kubectl get pods -n estudo \
  -o custom-columns='NAME:.metadata.name,APP:.metadata.labels.app,VERSION:.metadata.labels.version,NODE:.spec.nodeName'

# Filtrar por annotation via jsonpath (não é indexado, mas funciona)
kubectl get pods -n estudo -o json | \
  jq '.items[] | select(.metadata.annotations["prometheus.io/scrape"] == "true") | .metadata.name'
```

### Label selector em PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: webapp-pdb
  namespace: estudo
spec:
  minAvailable: 2          # sempre manter pelo menos 2 Pods
  selector:
    matchLabels:
      app: webapp
```

O PDB usa label selector para proteger os Pods corretos durante drains e upgrades.

### NetworkPolicy com namespaceSelector + podSelector

Labels em namespaces são usados para NetworkPolicy entre namespaces:

```bash
# Adicionar label ao namespace
kubectl label namespace databases app=databases tier=storage

# NetworkPolicy que usa o label do namespace
spec:
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: storage           # apenas de namespaces com este label
      podSelector:
        matchLabels:
          app: webapp             # e apenas Pods com este label
```

A separação `namespaceSelector` + `podSelector` no mesmo bloco `- from:` é um **AND** (namespace E pod). Em blocos separados seria um **OR**.

### Server-Side Apply e field ownership

Com `kubectl apply --server-side`, o apiserver rastreia quais campos pertencem a qual "field manager". Isso afeta labels e annotations — se dois processos tentam gerenciar o mesmo label, há conflito:

```bash
kubectl apply --server-side -f deployment.yaml
# Se outro field manager (ex: Helm) já tem ownership de um label:
# Apply failed with 1 conflict: conflict with "helm" using apps/v1
# → usar --force-conflicts para sobrescrever
kubectl apply --server-side --force-conflicts -f deployment.yaml
```

### Well-known labels do Kubernetes

O Kubernetes aplica automaticamente alguns labels em nodes e namespaces:

```bash
# Labels automáticos em nodes
kubectl get node k8s-worker1 --show-labels
# kubernetes.io/arch=amd64
# kubernetes.io/hostname=k8s-worker1
# kubernetes.io/os=linux
# node.kubernetes.io/exclude-from-external-load-balancers (em alguns setups)

# Labels automáticos em namespaces (K8s 1.21+)
kubectl get namespace estudo --show-labels
# kubernetes.io/metadata.name=estudo   ← usado em NetworkPolicy namespaceSelector
```

O label `kubernetes.io/metadata.name=<ns>` é fundamental para NetworkPolicy — é a forma correta de selecionar um namespace pelo nome:

```yaml
namespaceSelector:
  matchLabels:
    kubernetes.io/metadata.name: databases
```

---

## Referências

- [Labels e Selectors — Kubernetes Docs](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [Annotations — Kubernetes Docs](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)
- [Recommended Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)
- [Field Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/)
- [Well-Known Labels, Annotations and Taints](https://kubernetes.io/docs/reference/labels-annotations-taints/)
- Relacionados: [[05-Kubernetes/kubernetes-teoria-inicial]] | [[05-Kubernetes/services]] | [[05-Kubernetes/ingress]] | [[05-Kubernetes/networking-policy]]
