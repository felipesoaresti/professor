---
tags:
  - trilha
  - linux
area: linux
tipo: trilha
---

# Trilha Linux

> Canvas geral: [[trilha-devops-sre]]

## Assuntos Cobertos

| Assunto | Conteúdo | Exercícios | Status |
|---|---|---|---|
| LVM e Dispositivos de Disco | [[01-Linux/lvm-dispositivos-disco]] | [[11-Exercicios/lvm-dispositivos-disco]] | Exercícios gerados — pendente execução |
| Sistema de Arquivos | [[01-Linux/sistema-de-arquivos]] | [[11-Exercicios/sistema-de-arquivos]] | Exercícios gerados — pendente execução |
| Cgroups e Namespaces | [[01-Linux/cgroups-namespaces]] | [[11-Exercicios/cgroups-namespaces]] | Exercícios gerados — pendente execução |
| OOM Killer | [[01-Linux/oom-killer]] | [[11-Exercicios/oom-killer]] | Exercícios gerados — pendente execução |
| Processos e Scheduling | [[01-Linux/processos-scheduling]] | [[11-Exercicios/processos-scheduling]] | Exercícios gerados — pendente execução |

## Sequência Recomendada

### Fundamentos (começar aqui)
1. [[01-Linux/lvm-dispositivos-disco]] — Block devices, partições, LVM (PV/VG/LV)
2. [[01-Linux/sistema-de-arquivos]] — FHS, inodes, links, comandos de manipulação, troubleshooting
3. Processos e scheduling — fork, exec, CFS, load average *(a fazer)*
4. Memory management — virtual memory, page cache, swap *(a fazer)*

### Performance e Observabilidade
5. [[01-Linux/oom-killer]] — score, oom_score_adj, QoS K8s, cgroup OOM vs sistema OOM ✅
6. [[01-Linux/cgroups-namespaces]] — cgroups v1/v2, namespaces, containers internals ✅
7. [[01-Linux/processos-scheduling]] — fork/exec, estados, CFS, load average, zombies, affinity ✅
8. perf e flamegraphs — CPU profiling, stack traces *(a fazer)*
9. strace / ltrace — syscall tracing, debugging *(a fazer)*

### Avançado / Kernel
9. /proc e /sys — leitura sem ferramentas externas *(a fazer)*
10. ftrace e kprobes — kernel tracing *(a fazer)*

| rsync | [[01-Linux/rsync]] | [[11-Exercicios/rsync]] | Exercícios gerados — pendente execução |
| SSH / OpenSSH | [[01-Linux/ssh-openssh]] | [[11-Exercicios/ssh-openssh]] | [[12-Labs/ssh-openssh]] | Conteúdo gerado — pendente exercícios |

## Próximo Sugerido

**perf e flamegraphs** — agora que você entende CFS, cgroups e estados de processo, o passo natural é instrumentar CPU com `perf record` + flamegraph para ver onde o tempo realmente vai. Direto para troubleshooting de latência em produção.

Ou: **strace / ltrace** — rastreamento de syscalls com overhead cirúrgico, complemento direto do que foi visto em processos e scheduling.
