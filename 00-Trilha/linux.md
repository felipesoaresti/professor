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

## Sequência Recomendada

### Fundamentos (começar aqui)
1. [[01-Linux/lvm-dispositivos-disco]] — Block devices, partições, LVM (PV/VG/LV)
2. Filesystem internals — ext4, xfs, inodes, journaling *(a fazer)*
3. Processos e scheduling — fork, exec, CFS, load average *(a fazer)*
4. Memory management — virtual memory, page cache, swap *(a fazer)*

### Performance e Observabilidade
5. OOM killer — score, oom_adj, proteção de processos críticos *(a fazer)*
6. Cgroups v1/v2 — limites de CPU/memória, hierarquia *(a fazer)*
7. perf e flamegraphs — CPU profiling, stack traces *(a fazer)*
8. strace / ltrace — syscall tracing, debugging *(a fazer)*

### Avançado / Kernel
9. /proc e /sys — leitura sem ferramentas externas *(a fazer)*
10. ftrace e kprobes — kernel tracing *(a fazer)*
11. Containers internals — namespaces + cgroups na prática *(a fazer)*

## Próximo Sugerido

**Filesystem internals** — natural continuação após LVM: como ext4 e xfs gerenciam espaço, inodes, journaling. Crítico para entender o que acontece quando `df` e `du` divergem.

Ou: **OOM killer** — diretamente relevante para containers Kubernetes com memory limits.
