# Vault Professor — DevOps / SRE / Kubernetes

Base de conhecimento gerada pelo sistema de skills do Claude Code.

## Estrutura

| Pasta | Conteúdo |
|---|---|
| `00-Trilha/` | Trilhas de aprendizado por área (sequência recomendada de estudo) |
| `01-Linux/` | Kernel internals, performance, troubleshooting, /proc, strace, perf |
| `02-Networking/` | TCP/IP, DNS, HTTP, TLS, load balancing |
| `03-DevOps/` | CI/CD, GitOps, IaC, Terraform, automação |
| `04-SRE/` | SLI/SLO, error budget, incident management, golden signals |
| `05-Kubernetes/` | K8s completo — desde Pods até CKA-level |
| `06-Observabilidade/` | Prometheus, Grafana, PromQL, tracing, alerting |
| `07-Sistemas-Distribuidos/` | CAP, Raft, consensus, circuit breakers, idempotência |
| `08-Performance/` | CPU/memory/IO profiling, flamegraphs, bottleneck analysis |
| `09-Database/` | PostgreSQL, Redis — locks, replication, backup, tuning |
| `10-Cloud/` | AWS, OCI, GCP — VPC, IAM, storage, networking cloud |
| `11-Exercicios/` | Exercícios práticos sem resposta, organizados por assunto |
| `12-Labs/` | Labs guiados, livres e avançados |
| `13-Runbooks/` | Runbooks de troubleshooting e operação |
| `14-Bibliografia/` | Referências bibliográficas por domínio |
| `15-Notas/` | Anotações livres de aula |

## Como usar

Cada arquivo é gerado pela skill correspondente no Claude Code:

- `/professor <assunto>` — orchestrador, decide qual especialista usar
- `/professor-linux <assunto>`
- `/professor-networking <assunto>`
- `/professor-sre <assunto>`
- `/professor-devops <assunto>`
- `/professor-k8s <assunto>`
- `/professor-observabilidade <assunto>`
- `/professor-distributed <assunto>`

## Infraestrutura de Referência

### Kubernetes v1.35.3 — Debian 13 — Calico CNI

| Node | IP | Host |
|---|---|---|
| k8s-master | 192.168.3.30 | VM 300/pve2 |
| k8s-worker1 | 192.168.3.31 | VM 301/pve1 |
| k8s-worker2 | 192.168.3.32 | VM 302/pve2 |

Domínio: `*.staypuff.info` | Ingress: 192.168.3.100 (nginx/MetalLB)
Storage: `nfs-homelab` (retain) + `local-path` (ephemeral)

### Inventário Completo (verificado 2026-04-16)

| IP | Host | Descrição |
|---|---|---|
| 192.168.3.2 | pihole | DNS (LXC/pve1) |
| 192.168.3.3 | cf-tunnel | Cloudflare tunnel (LXC/pve1) |
| 192.168.3.4 | bastion | Jump host (LXC/pve1) |
| 192.168.3.5 | checkmk | Monitoramento (VM 304/pve2) |
| 192.168.3.6 | ubuntu | Ubuntu 24.04 (VM 305/pve1) |
| 192.168.3.20 | study | Debian 13 + Docker (VM 303/pve2) |
| 192.168.3.11 | pve1 | Proxmox host 1 |
| 192.168.3.12 | pve2 | Proxmox host 2 |
