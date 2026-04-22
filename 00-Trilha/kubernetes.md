---
tags:
  - trilha
  - kubernetes
area: kubernetes
tipo: trilha
---

# Trilha Kubernetes

> Canvas geral: [[trilha-devops-sre]]

## Como usar esta trilha

> [!info] Metodologia de aprendizado
> Cada módulo segue o mesmo ciclo. Não pule etapas — o feedback fecha o loop de aprendizado.

```
1. Leia o conteúdo teórico      → entenda o mecanismo interno, não só o comando
2. Rode os exemplos no homelab  → nada substitui executar no cluster real
3. Faça os exercícios           → sem consultar a solução; retorne com o que fez
4. Retorne ao professor         → revisão, correção e aprofundamento
5. Avance para o próximo        → só depois de executar os exercícios
```

> [!tip] Por que essa ordem importa
> Kubernetes tem dependências reais entre conceitos. Um StatefulSet sem entender PVC trava. RBAC sem entender ServiceAccount confunde. A sequência abaixo respeita essas dependências — não é arbitrária.

> [!warning] Não acumule teoria sem prática
> Ler 5 módulos sem executar nenhum exercício é o caminho mais rápido para esquecer tudo. Um módulo bem executado vale mais do que três lidos.

---

## Mapa de Progressão

| # | Estágio | Tópicos | Pré-requisito | Status |
|---|---------|---------|---------------|--------|
| 1 | **Fundamentos** | Teoria inicial, Labels e Annotations | Nenhum | ✅ |
| 2 | **Configuração** | ConfigMaps, Secrets | Estágio 1 | ✅ |
| 3 | **Workloads** | Deployment Strategies, Probes, DaemonSet, Jobs/CronJobs | Estágio 2 | ✅ |
| 4 | **Storage** | Volumes/StorageClass/PV/PVC, StatefulSet + Headless Service | Estágio 3 | ✅ |
| 5 | **Rede** | Services, Ingress, NetworkPolicy (Calico) | Estágio 4 | ✅ |
| 6 | **Segurança** | RBAC, SecurityContext, LimitRange/ResourceQuota | Estágio 5 | ❌ |
| 7 | **Scheduling** | NodeAffinity, PodAffinity/AntiAffinity, PDB, HPA/VPA | Estágio 6 | ❌ |
| 8 | **Operações** | kubectl avançado, Troubleshooting CKA-level, Cluster Lifecycle (kubeadm) | Estágios 1–7 | 🔄 |
| 9 | **GitOps** | Helm, Kustomize, ArgoCD | Estágio 8 | ❌ |

**Legenda:** ✅ conteúdo + exercícios criados · 🔄 conteúdo criado, exercícios pendentes de execução · ❌ a criar

---

## Sequência Detalhada

### Estágio 1 — Fundamentos (base obrigatória)

> [!info] Por que começar aqui
> Sem entender a arquitetura (control-plane, kubelet, etcd, API Server) e o modelo de labels/selectors, os outros recursos não fazem sentido. Labels são a cola que conecta Deployments a Services a NetworkPolicies.

| # | Módulo | Conteúdo | Exercícios | Esforço | Status |
|---|--------|----------|------------|---------|--------|
| 1.1 | Kubernetes — Teoria Inicial | [[05-Kubernetes/kubernetes-teoria-inicial]] | [[11-Exercicios/kubernetes-teoria-inicial]] | 3–4h | ✅ executado |
| 1.2 | Labels e Annotations | [[05-Kubernetes/labels-annotations]] | [[11-Exercicios/labels-annotations]] | 1–2h | 🔄 |

---

### Estágio 2 — Configuração

> [!info] Por que aqui
> ConfigMaps e Secrets são injetados em Pods via env vars ou volumes. Para entender Deployments avançados e StatefulSets, você precisa saber montar configuração de fora do container.

| # | Módulo | Conteúdo | Exercícios | Esforço | Status |
|---|--------|----------|------------|---------|--------|
| 2.1 | ConfigMaps | [[05-Kubernetes/configmaps]] | [[11-Exercicios/configmaps]] | 1–2h | 🔄 |
| 2.2 | Secrets | [[05-Kubernetes/secrets]] | [[11-Exercicios/secrets]] | 1–2h | 🔄 |

---

### Estágio 3 — Workloads

> [!info] Por que aqui
> Deployment Strategies dependem de Probes para funcionar corretamente — sem readiness probe, o RollingUpdate não sabe quando o Pod novo está pronto. DaemonSet e Jobs completam o catálogo de workloads antes de entrar em storage.

| # | Módulo | Conteúdo | Exercícios | Esforço | Status |
|---|--------|----------|------------|---------|--------|
| 3.1 | Deployment Strategies | [[05-Kubernetes/deployment-strategies]] | [[11-Exercicios/deployment-strategies]] | 2–3h | 🔄 |
| 3.2 | Probes (liveness, readiness, startup) | [[05-Kubernetes/probes]] | [[11-Exercicios/probes]] | 1–2h | 🔄 |
| 3.3 | DaemonSet | [[05-Kubernetes/daemonset]] | [[11-Exercicios/daemonset]] | 1–2h | 🔄 |
| 3.4 | Jobs e CronJobs | [[05-Kubernetes/jobs-cronjobs]] | [[11-Exercicios/jobs-cronjobs]] | 2h | 🔄 |

---

### Estágio 4 — Storage

> [!info] Por que aqui
> PV/PVC são pré-requisito direto para StatefulSet (volumeClaimTemplates). Só faz sentido estudar StatefulSet depois de entender StorageClass, ReclaimPolicy e os modos de acesso. No homelab: `local-path` para efêmero, `nfs-homelab` para persistente.

| # | Módulo | Conteúdo | Exercícios | Esforço | Status |
|---|--------|----------|------------|---------|--------|
| 4.1 | Volumes — StorageClass, PV, PVC | [[05-Kubernetes/volumes-storage]] | [[11-Exercicios/volumes-storage]] | 2–3h | 🔄 |
| 4.2 | StatefulSet e Headless Service | [[05-Kubernetes/statefulset-headless]] | [[11-Exercicios/statefulset-headless]] | 2–3h | 🔄 |

---

### Estágio 5 — Rede

> [!info] Por que aqui
> Services expõem Pods dentro e fora do cluster. Ingress roteia HTTP/HTTPS para Services. NetworkPolicy controla *quem pode falar com quem* — só faz sentido restringir tráfego depois de entender como ele flui. O homelab usa MetalLB + nginx-ingress + cert-manager + Calico.

| # | Módulo | Conteúdo | Exercícios | Esforço | Status |
|---|--------|----------|------------|---------|--------|
| 5.1 | Services | [[05-Kubernetes/services]] | [[11-Exercicios/services]] | 2h | 🔄 |
| 5.2 | Ingress | [[05-Kubernetes/ingress]] | [[11-Exercicios/ingress]] | 2–3h | 🔄 |
| 5.3 | NetworkPolicy (Calico) | [[05-Kubernetes/calico-network]] | [[11-Exercicios/calico-network]] | 2–3h | 🔄 |

---

### Estágio 6 — Segurança

> [!info] Por que aqui
> RBAC controla *quem pode fazer o quê* na API. SecurityContext define o que um container *pode fazer* no nó. ResourceQuota/LimitRange protegem o cluster de consumo excessivo. Esses três formam a base de hardening antes de colocar cargas de trabalho reais.

| # | Módulo | Conteúdo | Exercícios | Esforço | Status |
|---|--------|----------|------------|---------|--------|
| 6.1 | RBAC | *(a criar)* | *(a criar)* | 3–4h | ❌ |
| 6.2 | SecurityContext | *(a criar)* | *(a criar)* | 2h | ❌ |
| 6.3 | LimitRange e ResourceQuota | *(a criar)* | *(a criar)* | 2h | ❌ |

---

### Estágio 7 — Scheduling e Resiliência

> [!info] Por que aqui
> Affinity/AntiAffinity definem onde os Pods *devem* rodar. PodDisruptionBudget protege *quantos* ficam rodando durante drains. HPA/VPA escalam automaticamente. Os três juntos formam a base de resiliência operacional.

| # | Módulo | Conteúdo | Exercícios | Esforço | Status |
|---|--------|----------|------------|---------|--------|
| 7.1 | NodeAffinity e PodAffinity/AntiAffinity | *(a criar)* | *(a criar)* | 2–3h | ❌ |
| 7.2 | PodDisruptionBudget | *(a criar)* | *(a criar)* | 1–2h | ❌ |
| 7.3 | HPA e VPA | *(a criar)* | *(a criar)* | 2–3h | ❌ |

---

### Estágio 8 — Operações

> [!info] Por que aqui
> Troubleshooting e kubectl avançado requerem que você já tenha operado todos os recursos anteriores. Cluster Lifecycle (kubeadm) faz mais sentido depois de entender o que cada componente faz na prática.

| # | Módulo | Conteúdo | Exercícios | Esforço | Status |
|---|--------|----------|------------|---------|--------|
| 8.1 | Cluster Lifecycle (kubeadm, CNI, Kind) | [[05-Kubernetes/cluster-kubeadm-cni-kind]] | [[11-Exercicios/cluster-kubeadm-cni-kind]] | 3–4h | 🔄 |
| 8.2 | kubectl avançado | *(a criar)* | *(a criar)* | 2–3h | ❌ |
| 8.3 | Troubleshooting CKA-level | *(a criar)* | *(a criar)* | 3–4h | ❌ |

---

### Estágio 9 — GitOps

> [!info] Por que aqui
> Helm e Kustomize gerenciam manifests em escala. ArgoCD aplica GitOps — o estado do repo *é* o estado do cluster. Faz sentido apenas depois de dominar os recursos que você vai gerenciar com essas ferramentas.

| # | Módulo | Conteúdo | Exercícios | Esforço | Status |
|---|--------|----------|------------|---------|--------|
| 9.1 | Helm | *(a criar)* | *(a criar)* | 3–4h | ❌ |
| 9.2 | Kustomize | *(a criar)* | *(a criar)* | 2–3h | ❌ |
| 9.3 | ArgoCD | *(a criar)* | *(a criar)* | 3–4h | ❌ |

---

## Próximo Recomendado

Você concluiu os estágios 1–5 (conteúdo criado). Os exercícios dos estágios 2–5 estão **pendentes de execução**.

**Opção A — Executar exercícios pendentes** (recomendado antes de avançar)
Comece pelos exercícios de ConfigMaps → Secrets → Deployment Strategies → Probes. São os mais fundamentais e desencadeiam os outros.

**Opção B — Avançar para o Estágio 6: RBAC**
ServiceAccounts, Roles, ClusterRoles, RoleBindings, impersonation, audit log. Pré-requisito real para qualquer carga de trabalho segura em produção — e tema pesado no CKA.

**Opção C — Aprofundar Rede com NetworkPolicy**
O homelab usa Calico CNI. NetworkPolicy controla tráfego *dentro* do cluster. O Exercício 7 do cluster-kubeadm-cni-kind já introduz o conceito — agora é aplicar no cluster real com egress, ingress por namespace e GlobalNetworkPolicy do Calico.
