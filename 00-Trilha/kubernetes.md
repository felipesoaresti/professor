---
tags:
  - trilha
  - kubernetes
area: kubernetes
tipo: trilha
---

# Trilha Kubernetes

> Canvas geral: [[trilha-devops-sre]]

## Assuntos Cobertos

| Assunto | Conteúdo | Exercícios | Status |
|---|---|---|---|
| Kubernetes — Teoria Inicial | [[05-Kubernetes/kubernetes-teoria-inicial]] | [[11-Exercicios/kubernetes-teoria-inicial]] | Exercícios executados — ver logs em .txt |
| ConfigMaps | [[05-Kubernetes/configmaps]] | [[11-Exercicios/configmaps]] | Exercícios gerados — pendente execução |
| Deployment Strategies | [[05-Kubernetes/deployment-strategies]] | [[11-Exercicios/deployment-strategies]] | Exercícios gerados — pendente execução |
| Probes | [[05-Kubernetes/probes]] | [[11-Exercicios/probes]] | Exercícios gerados — pendente execução |
| DaemonSet | [[05-Kubernetes/daemonset]] | [[11-Exercicios/daemonset]] | Exercícios gerados — pendente execução |
| Volumes — StorageClass, PV, PVC | [[05-Kubernetes/volumes-storage]] | [[11-Exercicios/volumes-storage]] | Exercícios gerados — pendente execução |
| Secrets | [[05-Kubernetes/secrets]] | [[11-Exercicios/secrets]] | Exercícios gerados — pendente execução |
| StatefulSet e Headless Service | [[05-Kubernetes/statefulset-headless]] | [[11-Exercicios/statefulset-headless]] | Exercícios gerados — pendente execução |
| Services — ClusterIP, NodePort, LoadBalancer, kube-proxy | [[05-Kubernetes/services]] | [[11-Exercicios/services]] | Exercícios gerados — pendente execução |
| Ingress — nginx, Traefik, cert-manager, ClusterIssuer, TLS | [[05-Kubernetes/ingress]] | [[11-Exercicios/ingress]] | Exercícios gerados — pendente execução |
| Labels e Annotations — selectors, convenções, ferramentas | [[05-Kubernetes/labels-annotations]] | [[11-Exercicios/labels-annotations]] | Exercícios gerados — pendente execução |
| Cluster Lifecycle — kubeadm, CNI, Taints/Tolerations, Kind | [[05-Kubernetes/cluster-kubeadm-cni-kind]] | [[11-Exercicios/cluster-kubeadm-cni-kind]] | Exercícios gerados — pendente execução |
| Jobs e CronJobs — batch, indexed, parallel, TTL, migration pattern | [[05-Kubernetes/jobs-cronjobs]] | [[11-Exercicios/jobs-cronjobs]] | Exercícios gerados — pendente execução |

## Sequência Recomendada

### Fundamentos (base obrigatória)
1. [[05-Kubernetes/kubernetes-teoria-inicial]] — arquitetura, Pods, Deployments, Services, rollout/rollback ✅
2. [[05-Kubernetes/labels-annotations]] — labels, selectors, annotations, convenções app.kubernetes.io ✅
3. [[05-Kubernetes/configmaps]] — configuração desacoplada, variáveis de ambiente, volumes ✅
4. [[05-Kubernetes/deployment-strategies]] — RollingUpdate, Recreate, Blue-Green, Canary ✅

### Workloads e Storage
4. [[05-Kubernetes/secrets]] — Opaque, TLS, imagePullSecret, volume vs env var, RBAC, rotação ✅
5. [[05-Kubernetes/statefulset-headless]] — identidade estável, Headless Service, volumeClaimTemplates, partition ✅
6. [[05-Kubernetes/daemonset]] — por-node workloads, tolerations, hostPath, update strategy ✅
7. [[05-Kubernetes/volumes-storage]] — StorageClass, PV, PVC, local-path, nfs-homelab, ReclaimPolicy ✅

### Rede e Exposição
8. [[05-Kubernetes/services]] — ClusterIP, NodePort, LoadBalancer, ExternalName, kube-proxy, MetalLB ✅
9. [[05-Kubernetes/ingress]] — nginx, Traefik, cert-manager, ClusterIssuer/Issuer, staging vs prod, DNS-01 ✅
10. [[02-Networking/calico-network]] — BGP, Felix, IPAM, NetworkPolicy, GlobalNetworkPolicy, NetworkSet ✅

### Configuração e Segurança
11. RBAC — ServiceAccounts, Roles, ClusterRoles, RoleBindings *(a fazer)*
12. LimitRange e ResourceQuota — limites por namespace *(a fazer)*
13. SecurityContext — runAsNonRoot, readOnlyRootFilesystem, capabilities *(a fazer)*

### Cluster Lifecycle
14. [[05-Kubernetes/cluster-kubeadm-cni-kind]] — kubeadm fases, PKI, CNI (Flannel/Calico/Cilium), Taints/Tolerations, Kind ✅

### Workloads Batch
15. [[05-Kubernetes/jobs-cronjobs]] — Job, CronJob, Indexed Jobs, parallel patterns, migration pattern ✅

### Scheduling e Resiliência
16. NodeAffinity e PodAffinity/AntiAffinity — co-localização e anti-SPOF *(a fazer)*
15. PodDisruptionBudget — proteção durante deploys e drain *(a fazer)*
16. HPA e VPA — autoscaling horizontal e vertical *(a fazer)*

### Observabilidade e Operação
17. [[05-Kubernetes/probes]] — readiness, liveness, startup, shutdown gracioso ✅
18. kubectl avançado — jsonpath, custom-columns, patch, debug node *(a fazer)*
19. Troubleshooting CKA-level — metodologia sistemática de diagnóstico *(a fazer)*

### Avançado / GitOps
20. Helm — charts, values, upgrade, rollback *(a fazer)*
21. Kustomize — overlays, patches, bases *(a fazer)*
22. ArgoCD — GitOps, sync, app of apps *(a fazer)*

## Próximo Sugerido

**NetworkPolicy** — Ingress expõe serviços para fora. NetworkPolicy controla o tráfego *dentro* do cluster: quais Pods falam com quais, egress para namespaces externos, isolamento por namespace. O homelab usa Calico CNI — que suporta NetworkPolicy nativa e tem CRDs extras de GlobalNetworkPolicy. O Exercício 7 do cluster-kubeadm-cni-kind já introduz NetworkPolicy no Kind — agora é aprofundar no cluster real.

Ou: **RBAC** — ServiceAccounts, Roles, ClusterRoles, RoleBindings, impersonation, audit log.

Ou: **PodDisruptionBudget + HPA** — proteção durante drain (que apareceu no Exercício 8 do kubeadm) e autoscaling horizontal.
