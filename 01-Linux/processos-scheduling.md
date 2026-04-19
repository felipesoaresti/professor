---
tags:
  - linux
  - processos
  - scheduling
  - cfs
  - kernel
area: linux
tipo: conteudo
prerequisites:
  - "[[01-Linux/cgroups-namespaces]]"
next:
  - "[[11-Exercicios/processos-scheduling]]"
trilha: "[[00-Trilha/linux]]"
---

# Processos e Scheduling

> Conteúdo: [[01-Linux/processos-scheduling]] | Exercícios: [[11-Exercicios/processos-scheduling]] | Trilha: [[00-Trilha/linux]]

Entender processos e scheduling é entender por que o `load average` está em 40 com CPUs idle, por que um pod com `limits.cpu=500m` tem latência alta, por que um processo travou em D-state e não morre com `kill -9`. Todo troubleshooting de performance começa aqui.

---

## O que é e por que existe

O kernel Linux é um sistema operacional multitarefa preemptivo: múltiplos processos compartilham CPUs, e o kernel decide quem roda quando. O **scheduler** é o componente que toma essa decisão — qual processo roda em qual CPU, por quanto tempo, e em qual ordem.

O **processo** é a unidade fundamental de execução: tem espaço de endereçamento próprio, file descriptors, credenciais, e uma ou mais threads. O **thread** (task no kernel) é a unidade que o scheduler efetivamente agenda — cada thread tem sua própria pilha e registradores, mas compartilha o espaço de memória com as outras threads do mesmo processo.

---

## Como funciona internamente

### Ciclo de vida de um processo

```
fork() → clone() no kernel → nova task_struct
  ↓
exec() → carrega novo binário no mesmo espaço de endereçamento
  ↓
[execução, bloqueio, wakeup...]
  ↓
exit() → processo vira zombie (entra em estado Z)
  ↓
wait() pelo pai → kernel libera task_struct (processo desaparece)
```

**`fork()`** cria uma cópia exata do processo pai usando **Copy-on-Write (CoW)**: as páginas de memória não são copiadas imediatamente — pai e filho compartilham as mesmas páginas físicas com flag read-only. Quando qualquer um escreve, aí sim a página é copiada. Isso torna `fork()` quase instantâneo mesmo para processos grandes.

**`exec()`** descarta o espaço de endereçamento atual e carrega um novo binário. A combinação fork+exec é como todo processo é criado (shell scripts, daemons, containers).

### Estados de processo

```
R  Running/Runnable   — rodando ou na fila esperando CPU
S  Sleeping (interruptible)  — esperando evento, pode ser acordado por sinal
D  Sleeping (uninterruptible) — esperando I/O, NÃO pode ser interrompido por sinal
Z  Zombie             — terminou, aguardando wait() do pai
T  Stopped            — parado por SIGSTOP ou debugger (ptrace)
I  Idle               — kernel thread idle (não conta no load average)
```

```bash
# Ver estado de todos os processos
ps aux   # coluna STAT
ps -eo pid,stat,comm | grep "^[0-9]* D"  # processos em D-state

# Via /proc
cat /proc/<PID>/status | grep State
```

> [!warning] D-state e SIGKILL
> Um processo em D-state (uninterruptible sleep) **não pode ser morto** com `kill -9`. O SIGKILL é entregue, mas o processo só vai processá-lo quando sair do D-state. Se nunca sair (I/O travado, NFS pendente, bug de driver), o processo fica preso indefinidamente. Isso **conta no load average**.

### O que é load average de verdade

Load average (1m, 5m, 15m) **não é uso de CPU**. É a média do número de processos em estado **R + D** — processos que querem rodar (R) ou que estão bloqueados em I/O (D).

```
Load = processos_em_R + processos_em_D
```

- Load 4.0 em máquina com 4 CPUs → saturação exatamente no limite
- Load 4.0 em máquina com 8 CPUs → CPUs ainda têm folga
- Load 4.0 com CPUs idle → processos em D-state (I/O bound!) — CPUs livres mas processos esperando disco/rede/NFS

```bash
# Ver load average
uptime
cat /proc/loadavg
# 2.45 1.87 1.23  3/487  12345
# ↑1m  ↑5m  ↑15m  ↑procs_rodando/total  ↑último_pid

# Identificar contribuidores para load alto
ps -eo pid,stat,comm | grep " D "  # D-state contribui diretamente
```

### CFS — Completely Fair Scheduler

O CFS (desde Linux 2.6.23, substituiu o O(1) scheduler) usa uma **red-black tree** ordenada por `vruntime` (virtual runtime) — quanto tempo de CPU cada processo "consumiu" virtualmente.

**Invariante central:** o processo com menor `vruntime` é sempre o próximo a rodar.

```
vruntime += tempo_real_cpu * (peso_base / peso_processo)
```

O `peso_processo` vem do `nice` value:
- `nice -20` (mais alto) → peso maior → `vruntime` cresce mais devagar → mais CPU
- `nice 19` (mais baixo) → peso menor → `vruntime` cresce mais rápido → menos CPU

```bash
# Nice values e prioridade
nice -n -10 ./meu-processo    # executa com nice -10 (requer root para negativo)
renice -n 5 -p <PID>          # ajusta nice de processo em execução
ps -eo pid,ni,comm            # ver nice values
```

**Timeslice:** o CFS não tem timeslice fixo. O tempo que um processo roda antes de ser preemptado é proporcional ao seu peso relativo dentro do período de scheduling (padrão: 6ms a 24ms dependendo de quantos processos concorrem).

```bash
# Parâmetros do CFS
cat /proc/sys/kernel/sched_latency_ns       # período alvo (ns)
cat /proc/sys/kernel/sched_min_granularity_ns  # mínimo por timeslice
cat /proc/sys/kernel/sched_wakeup_granularity_ns
```

### Preemption

O Linux usa **preemption voluntária e involuntária**:

- **Voluntária:** processo chama `schedule()` explicitamente (quando faz I/O, `sleep()`, `mutex_lock()`)
- **Involuntária:** timer interrupt dispara, CFS verifica se outro processo deve rodar → preempta o atual

```bash
# Ver context switches (involuntários = preemptados, voluntários = bloquearam por conta)
cat /proc/<PID>/status | grep ctxt_switches
# voluntary_ctxt_switches: 12345    → processo dormiu/esperou I/O
# nonvoluntary_ctxt_switches: 67    → foi preemptado pelo scheduler

# Se nonvoluntary é alto → processo CPU-bound sendo preemptado frequentemente
```

### Scheduling classes e prioridades

O kernel tem múltiplas scheduling classes em ordem de prioridade:

```
1. SCHED_DEADLINE  — real-time, garante prazo (EDF)
2. SCHED_FIFO      — real-time, FIFO dentro da prioridade (0-99)
3. SCHED_RR        — real-time, round-robin (0-99)
4. SCHED_OTHER/NORMAL — CFS padrão (nice -20 a 19)
5. SCHED_BATCH     — batch, penaliza wakeup latency
6. SCHED_IDLE      — só roda quando não há mais nada
```

```bash
# Ver e alterar scheduling class de um processo
chrt -p <PID>   # ver política atual
chrt -f -p 50 <PID>  # SCHED_FIFO prioridade 50 (cuidado em produção)
chrt -o -p 0 <PID>   # voltar para SCHED_OTHER
```

> [!warning] Processos real-time (FIFO/RR) podem starvar o sistema
> Um processo `SCHED_FIFO` com prioridade alta roda até voluntariamente ceder a CPU. Se travar em loop infinito, pode starvar todos os outros processos, incluindo o scheduler. Nunca use SCHED_FIFO em produção sem entender o impacto.

### CPU affinity

```bash
# Ver quais CPUs um processo pode usar
taskset -p <PID>
# pid 1234's current affinity mask: f → 0b1111 → CPUs 0,1,2,3

# Restringir processo às CPUs 0 e 1
taskset -pc 0,1 <PID>

# Executar novo processo restrito à CPU 2
taskset -c 2 ./meu-processo

# Via /proc
cat /proc/<PID>/status | grep Cpus_allowed
```

---

## Na prática — comandos e exemplos reais

### Inspecionando processos via /proc

```bash
# Status completo de um processo
cat /proc/<PID>/status
# Name, State, Pid, PPid, Threads, VmRSS, VmSize, etc.

# Estatísticas de scheduling (números brutos do kernel)
cat /proc/<PID>/sched
# nr_switches, nr_voluntary_switches, wait_sum (tempo esperando CPU em ns)

# Linha de comando completa (com argumentos)
cat /proc/<PID>/cmdline | tr '\0' ' '

# Environment variables do processo
cat /proc/<PID>/environ | tr '\0' '\n'

# File descriptors abertos
ls -la /proc/<PID>/fd
ls /proc/<PID>/fd | wc -l  # contar fds (checar leak)
```

### Monitorar scheduling com perf

```bash
# Context switches por segundo no sistema
perf stat -e cs sleep 10

# Context switches por processo
perf stat -e cs -p <PID> sleep 10

# Quem está usando mais CPU (sampler)
perf top -e cpu-clock

# Ver migração de CPU (processo sendo movido entre CPUs)
perf stat -e migrations -p <PID> sleep 10
```

### Zombie processes — identificar e resolver

```bash
# Listar zombies
ps aux | grep " Z "
ps -eo pid,ppid,stat,comm | awk '$3 ~ /Z/'

# Identificar o pai do zombie
cat /proc/<ZOMBIE-PID>/status | grep PPid

# Solução: matar o pai (ele vai chamar wait() antes de morrer)
# ou: enviar SIGCHLD ao pai para que ele faça wait()
kill -CHLD <PPID>

# Se o pai for problemático, matar o pai
kill <PPID>
# O zombie passa para o init (PID 1) que chama wait() automaticamente
```

> [!info] Zombie não consome recursos
> Um processo zombie não consome CPU nem memória (espaço de endereçamento já foi liberado). O que ele ocupa é apenas uma entrada na tabela de processos e um PID. O problema é quando acumulam muitos zombies — pode esgotar a tabela de PIDs (`/proc/sys/kernel/pid_max`).

### Analisando CPU steal em VMs

```bash
# CPU steal = tempo que o kernel queria rodar mas o hypervisor negou a CPU
# Visível em: vmstat, top, sar
vmstat 1 10
# coluna "st" = steal time

top
# Linha "%Cpu(s)": us sy ni id wa hi si st
#                                          ↑ steal

# Alto steal → host físico overcomprometido → problema do provedor cloud / hipervisor
# Sintoma: load normal, CPU us baixa, mas latência alta
```

### Rastreando fork/exec com strace

```bash
# Rastrear todos os processos filhos (fork + exec)
strace -f -e trace=fork,vfork,clone,execve -p <PID>

# Ver apenas exec (o que está sendo executado)
strace -f -e execve <comando>

# Tempo gasto em cada syscall
strace -c <comando>
```

---

## Casos de uso e boas práticas

### kubectl top vs. load average no node

```bash
# kubectl top node mostra CPU usage (%) — não load average
# Load average alto com CPU% baixo no top → processos em D-state (I/O)

# No node, identificar a causa:
iostat -x 1 5   # await alto em disco = D-state por I/O de disco
nfsstat         # se há NFS montado, checar timeouts
dmesg | grep -i "hung_task\|blocked for"  # kernel detecta D-state longo
```

### CPU requests como "peso" no CFS

O `requests.cpu` do Kubernetes se traduz diretamente em `cpu.weight` no cgroup:

```
weight = (1024 * requests_millicores) / 1000
```

Em um node com 2 pods — um com `requests.cpu=1000m` e outro com `requests.cpu=500m`:
- Pod A tem peso ~1024
- Pod B tem peso ~512
- Sob contenção, Pod A recebe ~66% da CPU e Pod B ~33%

```bash
# Ver o peso cgroup de um pod
CGROUP=$(cat /proc/<PID>/cgroup | cut -d: -f3)
cat /sys/fs/cgroup${CGROUP}/cpu.weight
```

### Evitar que um processo crítico seja preemptado

```bash
# Para processos críticos de baixa latência: aumentar nice negativamente
renice -n -10 -p <PID>

# Para realmente reduzir preemption: SCHED_BATCH (reduz interrupções)
chrt -b -p 0 <PID>

# Isolar CPU inteiramente para um processo (isolcpus no boot)
# Em /etc/default/grub: GRUB_CMDLINE_LINUX="isolcpus=3"
# Depois: taskset -c 3 ./processo-critico
```

---

## Troubleshooting — cenários reais de produção

### Cenário 1: Load average alto, CPUs idle

```bash
# 1. Confirmar o diagnóstico
uptime         # load alto
mpstat 1 5     # CPUs idle (%idle alto)

# 2. Identificar processos em D-state
ps -eo pid,stat,comm,wchan | grep " D "
# wchan = nome da função do kernel onde o processo está dormindo

# 3. Ver qual syscall está bloqueando
cat /proc/<PID>/wchan      # nome da função
cat /proc/<PID>/stack      # stack completo do kernel (onde exatamente travou)

# 4. Causas comuns de D-state longo:
# - NFS com servidor indisponível
# - Disco com falha (await muito alto no iostat)
# - Lock de filesystem
# - Bug de driver
```

### Cenário 2: Processo "travado" que não responde a SIGKILL

```bash
# 1. Verificar estado
cat /proc/<PID>/status | grep State
# State: D (disk sleep) → D-state, não vai responder

# 2. Ver onde travou no kernel
cat /proc/<PID>/wchan
cat /proc/<PID>/stack

# 3. Opções (piores caso):
# - Aguardar I/O resolver (se for timeout de NFS, vai destravar)
# - Se for hardware, reboot pode ser necessário
# - dmesg | grep "hung task" para ver se o kernel já detectou

# 4. Kernel detecta automaticamente após 120s (padrão)
cat /proc/sys/kernel/hung_task_timeout_secs
```

### Cenário 3: Pod com CPU usage errático (spike + idle)

```bash
# 1. Verificar se é throttling (limits muito baixos)
CGROUP=$(cat /proc/<PID>/cgroup | cut -d: -f3)
cat /sys/fs/cgroup${CGROUP}/cpu.stat | grep throttled

# 2. Verificar context switches (processo sendo preemptado)
watch -n1 "cat /proc/<PID>/status | grep ctxt"

# 3. Verificar se o processo migra entre CPUs (cache miss)
perf stat -e migrations -p <PID> sleep 10

# 4. Verificar se há CPU steal (problema de hypervisor)
vmstat 1 10 | awk '{print $16}'  # coluna st
```

### Cenário 4: Fork bomb — muitos processos filhos

```bash
# Detectar
cat /proc/sys/kernel/pid_max    # limite de PIDs
cat /proc/sys/kernel/threads-max  # limite de threads

# Contagem atual
ls /proc | grep -c '^[0-9]'

# Limitar com cgroup pids (antes que aconteça)
cat /sys/fs/cgroup/<grupo>/pids.max
echo 100 > /sys/fs/cgroup/<grupo>/pids.max
# Kubernetes expõe isso via spec.containers[].resources.limits.pids (1.28+)

# Limitar com ulimit (por usuário)
ulimit -u 1000  # máximo 1000 processos para o usuário atual
```

---

## Nível avançado — kernel internals, CKA/Staff

### task_struct — a estrutura de processo no kernel

Cada processo/thread é representado por uma `task_struct` em `include/linux/sched.h`. Campos relevantes:

```c
struct task_struct {
    volatile long state;     // estado: TASK_RUNNING, TASK_INTERRUPTIBLE, etc.
    pid_t pid;               // PID desta thread
    pid_t tgid;              // thread group ID = PID do processo pai de threads
    struct task_struct *parent;
    struct list_head children;
    struct sched_entity se;  // entidade CFS (vruntime, etc.)
    int prio;                // prioridade dinâmica
    int static_prio;         // prioridade estática (nice convertido)
    struct nsproxy *nsproxy; // namespaces
    struct css_set *cgroups; // cgroups
    // ... centenas de outros campos
};
```

```bash
# Acessar campos da task_struct via /proc sem ferramentas
cat /proc/<PID>/stat | cut -d' ' -f1-5
# PID comm state PPID pgrp ...
```

### vruntime e fairness

O CFS garante fairness através do `vruntime`. Quando um processo acorda após longa espera, seu `vruntime` pode estar muito atrás — o CFS clampeia em `min_vruntime - latency_target` para evitar que ele monopolize a CPU tentando "recuperar" o tempo perdido.

```bash
# Ver vruntime e outras métricas de scheduling de um processo
cat /proc/<PID>/sched
# se_sum_exec_runtime → tempo total de CPU em ns
# wait_sum            → tempo total esperando na runqueue em ns
# nr_switches         → total de context switches
```

### Scheduler groups e CPU bandwidth no CFS

O CFS organiza tarefas em grupos hierárquicos alinhados com a hierarquia de cgroups:

```
CPU 0 runqueue
  ├── cfs_rq (raiz)
  │   ├── task A (vruntime=100)
  │   └── sched_group: kubepods.slice
  │       ├── cfs_rq (grupo)
  │       │   ├── task B (container 1)
  │       │   └── task C (container 1)
  │       └── cfs_rq (grupo)
  │           └── task D (container 2)
```

O throttling de CPU (`cpu.max`) é implementado pelo **CFS bandwidth controller**: cada grupo tem um "token bucket" de quota. Quando a quota se esgota no período, o sched_group inteiro é colocado em throttled state.

### Numa topology e scheduling

Em servidores com múltiplos sockets NUMA:

```bash
# Ver topologia NUMA
numactl --hardware
numactl --show  # NUMA node do processo atual

# Ver afinidade NUMA de um processo
cat /proc/<PID>/status | grep Mems_allowed

# Rodar processo com afinidade NUMA
numactl --cpunodebind=0 --membind=0 ./processo

# Scheduler tenta manter processos no mesmo NUMA node (NUMA-aware scheduling)
# Monitorar migração NUMA:
cat /proc/<PID>/sched | grep numa_migrations
```

### ftrace para analisar scheduling

```bash
# Habilitar tracer de scheduling
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_wakeup/enable

# Capturar eventos
cat /sys/kernel/debug/tracing/trace | head -100

# Filtrar por PID específico
echo "common_pid == <PID>" > /sys/kernel/debug/tracing/events/sched/sched_switch/filter

# Desabilitar
echo 0 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
echo 0 > /sys/kernel/debug/tracing/events/sched/sched_wakeup/enable
```

---

## Referências

- **Brendan Gregg** — "Systems Performance" cap. 6 (CPUs), "BPF Performance Tools" cap. 6
- **Robert Love** — "Linux Kernel Development" cap. 3 e 4 (Process Management, Scheduling)
- **Kernel docs** — [CFS Scheduler](https://www.kernel.org/doc/html/latest/scheduler/sched-design-CFS.html)
- **Linuxtips DK8S** — `/mnt/c/Users/felip/Documents/Kubernets/DK8S`
- `man 5 proc` — descrição completa de `/proc/PID/`
- `man 7 sched` — scheduling policies e prioridades
- Brendan Gregg flamegraph scripts — github.com/brendangregg/FlameGraph
