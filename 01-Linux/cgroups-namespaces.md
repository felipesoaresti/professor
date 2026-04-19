---
tags:
  - linux
  - cgroups
  - namespaces
  - containers
  - kernel
area: linux
tipo: conteudo
prerequisites:
  - "[[01-Linux/sistema-de-arquivos]]"
  - "[[01-Linux/lvm-dispositivos-disco]]"
next:
  - "[[11-Exercicios/cgroups-namespaces]]"
trilha: "[[00-Trilha/linux]]"
---

# Cgroups e Namespaces

> Conteúdo: [[01-Linux/cgroups-namespaces]] | Exercícios: [[11-Exercicios/cgroups-namespaces]] | Trilha: [[00-Trilha/linux]]

Esses dois mecanismos do kernel são os alicerces de toda containerização moderna. **Namespaces** isolam o que um processo *vê*. **Cgroups** controlam o que um processo *pode usar*. Juntos, formam o que chamamos de container — sem virtualização de hardware.

---

## O que é e por que existe

### Namespaces

Antes dos namespaces (antes do Linux 3.8), todo processo no sistema compartilhava a mesma visão global: mesma tabela de PIDs, mesma stack de rede, mesmo hostname. Não havia forma nativa de isolar processos uns dos outros sem VMs.

O namespace é uma abstração do kernel que cria uma *cópia privada* de um recurso global. Um processo dentro de um namespace vê apenas os recursos daquele namespace — não os do host nem de outros namespaces.

### Cgroups

Control Groups (cgroups) foram criados pelo Google em 2006 (Paul Menage e Rohit Seth) e mergeados no kernel 2.6.24 (2008). O problema que resolviam: como garantir que um processo mal comportado não consuma todo o CPU/memória do sistema e afete outros?

Cgroups criam uma hierarquia de *grupos de processos* e permitem aplicar políticas de resource accounting e limiting a cada grupo. Toda a telemetria de uso (`kubectl top`, `docker stats`) vem do accounting dos cgroups.

---

## Como funciona internamente

### Namespaces — os 8 tipos

| Namespace | Flag `clone()` | Isola | Visível em |
|---|---|---|---|
| **mnt** | `CLONE_NEWNS` | Filesystem mounts (árvore de mount points) | `/proc/PID/mounts` |
| **pid** | `CLONE_NEWPID` | Árvore de PIDs (PID 1 dentro do ns) | `/proc/PID/status` (NSpid) |
| **net** | `CLONE_NEWNET` | Stack de rede (interfaces, rotas, iptables) | `/proc/PID/net/` |
| **uts** | `CLONE_NEWUTS` | Hostname e domainname | `uname -n` dentro do ns |
| **ipc** | `CLONE_NEWIPC` | Shared memory, semáforos, message queues | `/proc/PID/status` |
| **user** | `CLONE_NEWUSER` | UIDs/GIDs (root dentro ≠ root fora) | `/proc/PID/uid_map` |
| **cgroup** | `CLONE_NEWCGROUP` | Visão da hierarquia cgroup (vê raiz falsa) | `/proc/PID/cgroup` |
| **time** | `CLONE_NEWTIME` | Clock offsets (Linux 5.6+) | `/proc/PID/timens_offsets` |

**Como o kernel implementa:** cada processo tem no `task_struct` uma referência para um `nsproxy` — uma struct que aponta para o namespace de cada tipo ao qual o processo pertence. `fork()` + `CLONE_NEW*` cria novos namespaces e atualiza o `nsproxy` do filho.

```
task_struct
  └── nsproxy
        ├── mnt_ns   → mount namespace
        ├── pid_ns   → PID namespace
        ├── net_ns   → network namespace
        ├── ipc_ns   → IPC namespace
        ├── uts_ns   → UTS namespace
        └── user_ns  → user namespace
```

### Cgroups v1 vs v2

**v1 (legado, ainda presente):**
- Cada controller (cpu, memory, blkio, pids…) tem sua própria hierarquia independente
- Um processo pode estar em hierarquias diferentes para cada controller
- Montado em `/sys/fs/cgroup/<controller>/`
- Problema: hierarquias inconsistentes, race conditions no memory + swap accounting

**v2 (unified hierarchy, padrão desde kernel 4.5, default em Debian 11+):**
- Uma hierarquia única para todos os controllers
- Cada processo pertence a exatamente um lugar na hierarquia
- Montado em `/sys/fs/cgroup/` (sem subdiretório por controller)
- Interface mais consistente, sem ambiguidades

```bash
# Verificar qual versão está ativa
mount | grep cgroup
# cgroup2 on /sys/fs/cgroup type cgroup2 → v2 puro
# cgroup on /sys/fs/cgroup/memory type cgroup → v1
```

> [!info] Debian 13 (trixie) + containerd 2.2.2
> O homelab roda cgroup v2 puro. Todos os exemplos abaixo focam em v2, mas mostram equivalentes v1 onde relevante.

### Hierarquia cgroup v2

```
/sys/fs/cgroup/
├── cgroup.controllers          # controllers disponíveis no root
├── cgroup.subtree_control      # controllers delegados aos filhos
├── cpu.stat                    # accounting do root cgroup
├── memory.current              # memória total do sistema vista pelo cgroup
├── system.slice/               # serviços systemd
│   ├── containerd.service/
│   └── ssh.service/
├── kubepods.slice/             # todos os pods Kubernetes
│   ├── kubepods-burstable.slice/
│   │   └── kubepods-burstable-pod<uid>.slice/
│   │       ├── cri-containerd-<containerid>.scope/
│   │       │   ├── cpu.max          # limite de CPU
│   │       │   ├── memory.max       # limite de memória
│   │       │   └── pids.max         # limite de PIDs
│   └── kubepods-besteffort.slice/
└── user.slice/
```

### Como um container é criado (runc/containerd)

1. `containerd` recebe request do kubelet (via CRI gRPC)
2. Cria um cgroup scope em `/sys/fs/cgroup/kubepods.slice/.../<container-id>.scope`
3. Escreve os limites nos arquivos do cgroup (`cpu.max`, `memory.max`, `pids.max`)
4. Chama `runc` que executa `clone()` com todos os `CLONE_NEW*` flags
5. O processo filho está agora em namespaces isolados E dentro do cgroup configurado
6. `runc` faz pivot_root (ou chroot) para o rootfs do container (namespace mnt)
7. Executa o `entrypoint` do container como PID 1 (dentro do pid namespace)

---

## Na prática — comandos e exemplos reais

### Inspecionando namespaces

```bash
# Listar todos os namespaces do sistema
lsns

# Listar namespaces de um processo específico
lsns -p <PID>

# Ver os namespaces de um processo via /proc
ls -la /proc/<PID>/ns/

# Comparar se dois processos compartilham namespace (mesmo inode = mesmo ns)
ls -lai /proc/1/ns/net /proc/<container-PID>/ns/net
```

```bash
# Entrar no namespace de rede de um container (sem docker exec)
# Encontrar PID do processo no host
crictl inspect <container-id> | grep pid
# ou
cat /proc/<PID>/status | grep NSpid

# Entrar no net namespace
nsenter -t <PID> --net ip addr
nsenter -t <PID> --net ss -tulnp

# Entrar em TODOS os namespaces (equivale a exec no container)
nsenter -t <PID> --mount --uts --ipc --net --pid -- bash
```

```bash
# Criar um novo namespace de rede isolado manualmente
ip netns add meu-ns
ip netns exec meu-ns ip addr
ip netns exec meu-ns bash

# unshare — executa comando em novos namespaces
unshare --pid --fork --mount-proc bash
# Agora você é PID 1 nesse namespace!
ps aux   # vê apenas seu processo
```

### Inspecionando cgroups v2

```bash
# Ver a qual cgroup um processo pertence
cat /proc/<PID>/cgroup
# Exemplo de saída:
# 0::/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-podXXX.slice/cri-containerd-YYY.scope

# Navegar até o cgroup do container
CGROUP=$(cat /proc/<PID>/cgroup | cut -d: -f3)
ls /sys/fs/cgroup${CGROUP}

# Ver limite de CPU
cat /sys/fs/cgroup${CGROUP}/cpu.max
# Saída: 100000 100000  → (quota μs) (period μs) → 100% de 1 core
# Saída: 50000 100000   → 50% de 1 core
# Saída: max 100000     → sem limite

# Ver uso atual de CPU
cat /sys/fs/cgroup${CGROUP}/cpu.stat

# Ver limite de memória
cat /sys/fs/cgroup${CGROUP}/memory.max
# max = sem limite; número em bytes = limite

# Ver uso atual de memória
cat /sys/fs/cgroup${CGROUP}/memory.current

# Ver throttling de CPU (campo crítico!)
cat /sys/fs/cgroup${CGROUP}/cpu.stat | grep throttled
# nr_throttled   → quantas vezes o processo foi pausado
# throttled_usec → tempo total parado por throttling
```

```bash
# systemd-cgls — visão em árvore bonita
systemd-cgls

# systemd-cgtop — top por cgroup (live)
systemd-cgtop

# Encontrar qual cgroup tem mais memória em uso
for d in /sys/fs/cgroup/kubepods.slice/**/memory.current; do
  echo "$d: $(cat $d)"
done | sort -t: -k2 -n | tail -20
```

### Criando cgroups manualmente (v2)

```bash
# Criar um cgroup filho
mkdir /sys/fs/cgroup/meu-grupo

# Habilitar controllers
echo "+cpu +memory +pids" > /sys/fs/cgroup/cgroup.subtree_control
echo "+cpu +memory +pids" > /sys/fs/cgroup/meu-grupo/cgroup.subtree_control

# Definir limite de memória (100MB)
echo $((100 * 1024 * 1024)) > /sys/fs/cgroup/meu-grupo/memory.max

# Definir limite de CPU (50% de 1 core)
echo "50000 100000" > /sys/fs/cgroup/meu-grupo/cpu.max

# Mover processo para o cgroup
echo <PID> > /sys/fs/cgroup/meu-grupo/cgroup.procs

# Verificar
cat /proc/<PID>/cgroup
```

---

## Casos de uso e boas práticas

### CPU requests e limits em Kubernetes — o que acontece de verdade

```
spec.resources.requests.cpu: "250m"
spec.resources.limits.cpu: "500m"
```

- **requests** → usado pelo scheduler para decidir onde colocar o pod. Define o `cpu.weight` no cgroup (share proporcional de CPU quando há contenção)
- **limits** → vira `cpu.max` no cgroup. Implementa CPU throttling via Completely Fair Scheduler (CFS bandwidth controller)

> [!warning] CPU Throttling é silencioso e devastador
> Um pod com `limits.cpu: 500m` em uma máquina idle vai ser *throttled* mesmo sem concorrência. O kernel pausa o processo quando a quota do período se esgota. Sintoma: latência de p99 alta sem CPU usage alto. Diagnóstico: `cpu.stat` → `throttled_usec`.

```bash
# Verificar throttling em todos os pods do cluster (no node)
for scope in /sys/fs/cgroup/kubepods.slice/**/*.scope; do
  throttled=$(grep throttled_usec $scope/cpu.stat 2>/dev/null | awk '{print $2}')
  [ "$throttled" -gt 0 ] 2>/dev/null && echo "$scope: ${throttled}μs throttled"
done
```

### Memory limits — OOM no cgroup vs OOM do kernel

Quando um processo excede `memory.max`:
1. O kernel tenta reclamar memória (page cache, swap se configurado)
2. Se não conseguir → `memory.oom_kill` dispara
3. O processo é killed dentro do cgroup (não o sistema todo)
4. No Kubernetes: Pod reinicia com `OOMKilled` no status

```bash
# Ver se houve OOM kills em um cgroup
cat /sys/fs/cgroup${CGROUP}/memory.events
# oom          → quantas vezes oom foi ativado
# oom_kill     → quantas vezes matou processo
```

> [!tip] memory.high vs memory.max
> - `memory.max` = hard limit → OOM killer
> - `memory.high` = soft limit → kernel throttle memory allocation, mas não mata
> Kubernetes usa apenas `memory.max` (limits). Não há equivalente nativo a `memory.high` em K8s resources.

---

## Troubleshooting — cenários reais de produção

### Cenário 1: "Pod lento, CPU usage está baixo no Grafana"

**Hipótese:** CPU throttling pelo cgroup — o pod está sendo pausado pelo kernel mesmo com CPU aparentemente livre.

```bash
# No node onde o pod está rodando
# 1. Encontrar o PID do container
crictl ps | grep <nome-do-pod>
crictl inspect <container-id> | grep '"pid"'

# 2. Encontrar o cgroup
cat /proc/<PID>/cgroup

# 3. Verificar throttling
CGROUP_PATH=$(cat /proc/<PID>/cgroup | cut -d: -f3)
watch -n1 cat /sys/fs/cgroup${CGROUP_PATH}/cpu.stat

# Se nr_throttled aumenta → é throttling, aumente o limits.cpu
```

### Cenário 2: "Container não vê o hostname correto"

**Causa:** o processo está no UTS namespace isolado do container, mas algo modificou `/etc/hostname` no host.

```bash
# Verificar hostname dentro vs fora
nsenter -t <PID> --uts -- hostname
hostname  # no host
# São diferentes — correto para containers
```

### Cenário 3: "Processo fugiu do cgroup" / "kubectl top mostra errado"

```bash
# Verificar se todos os processos do container estão no cgroup correto
cat /sys/fs/cgroup${CGROUP}/cgroup.procs
# Lista de PIDs — deve bater com os processos do container

# Processos filhos herdam o cgroup — se um processo faz fork e o filho
# não aparece aqui, pode indicar namespace leak ou bug no runtime
```

### Cenário 4: Investigar rede de container sem kubectl exec

```bash
# Às vezes kubectl exec não está disponível (pod crashando, imagem sem shell)
# Use nsenter do host

# Encontrar PID
crictl inspect <id> | grep '"pid"'

# Inspecionar rede
nsenter -t <PID> --net ss -tulnp
nsenter -t <PID> --net ip route
nsenter -t <PID> --net iptables -L -n  # se tiver permissão
nsenter -t <PID> --net tcpdump -i eth0 -n  # captura dentro do ns
```

---

## Nível avançado — edge cases, kernel internals, Staff-level

### PID namespace e o problema do PID 1

Dentro de um PID namespace, o primeiro processo é o PID 1. O kernel trata PID 1 de forma especial:
- Não recebe SIGKILL por padrão (só de dentro do mesmo namespace ou do namespace pai)
- Quando PID 1 morre, *todos os outros processos do namespace recebem SIGKILL*
- Isso é o comportamento de "container restart" — o init process morreu

> [!warning] Sinal SIGTERM em containers
> Se seu container não faz trap de SIGTERM no PID 1, ele não vai desligar graciosamente. O `kubectl drain` manda SIGTERM → espera `terminationGracePeriodSeconds` → manda SIGKILL. Se o processo não responde ao SIGTERM (por não ser PID 1 real, por exemplo via shell script wrapper), você sempre vai ter shutdown abrupto.

```bash
# Ver o PID real no host de um processo que é PID 1 dentro do container
cat /proc/<HOST-PID>/status | grep -E "^(Pid|NSpid)"
# Pid: 12345         → PID no host
# NSpid: 12345  1    → PIDs em cada namespace aninhado (o 1 é o PID dentro do container)
```

### User namespace e rootless containers

Com user namespaces, um processo pode ser UID 0 (root) dentro do namespace mas mapeado para um UID não-privilegiado no host:

```bash
# Ver o mapeamento de UIDs
cat /proc/<PID>/uid_map
# dentro_ns  fora_ns  tamanho
# 0          1000     1   → UID 0 dentro = UID 1000 fora (apenas 1 UID mapeado)
# 0          100000   65536 → root dentro = 100000 fora (subuid range)
```

Rootless containers (Podman, rootless Docker) usam isso extensivamente. O containerd do K8s em geral não usa user namespaces ainda (suporte experimental em 1.25+, `hostUsers: false`).

### cgroup v2 e o delegated model do systemd

O systemd atua como "cgroup manager" — ele controla o cgroup root e delega subárvores para serviços. Containerd recebe uma subárvore delegada e cria scopes dentro dela para cada container.

```bash
# Ver controllers habilitados em cada nível da hierarquia
cat /sys/fs/cgroup/cgroup.subtree_control
cat /sys/fs/cgroup/kubepods.slice/cgroup.subtree_control
# Se um controller não aparece aqui, não pode ser usado pelos filhos
```

> [!warning] kubelet cgroup driver
> O kubelet precisa saber qual cgroup manager usar: `cgroupDriver: systemd` ou `cgroupDriver: cgroupfs`. Se houver mismatch entre kubelet e containerd, os pods crasham silenciosamente. Verificar: `kubectl get node -o yaml | grep cgroupDriver` e `cat /etc/containerd/config.toml | grep SystemdCgroup`.

### CPU Bandwidth Controller — como o throttling funciona

O CFS bandwidth controller opera em períodos (padrão: 100ms). Cada cgroup tem uma quota:
- `cpu.max = Q T` → o cgroup pode usar até Q μs por período de T μs
- Se o cgroup usar a quota antes do período acabar → todos os processos são colocados em `THROTTLED` state até o próximo período
- O período de reset é implementado por um timer do kernel

```
Período: |----100ms----|----100ms----|
Quota:    [==50ms==]                    → processo para aqui
          [==50ms==]  run normal...
                throttled! ^^
```

```bash
# Monitorar throttling em tempo real
while true; do
  date
  cat /sys/fs/cgroup${CGROUP}/cpu.stat | grep -E "throttled|nr_periods"
  sleep 1
done
```

### Namespace leaks e como detectar

```bash
# Listar todos os net namespaces do sistema
lsns -t net

# Se um namespace persiste após o container morrer, há um leak
# Causas: processo que saiu do container mas mantém fd aberto no /proc/PID/ns/net
# Ou: bind mount de namespace (ip netns add faz isso em /run/netns/)

# Cleanup de net namespaces órfãos
ip netns list
ip netns delete <nome>

# Para namespaces sem nome (anonymous), identificar pelo inode
ls -lai /proc/*/ns/net | sort -k1 | uniq -d -f0
```

### cgroup memory e o Slab cache problem

```bash
# memory.current inclui page cache e slab cache!
# Um pod pode parecer usar muita memória mas boa parte é cache recuperável

cat /sys/fs/cgroup${CGROUP}/memory.stat
# file          → page cache (recuperável)
# anon          → memória anônima (não recuperável sem swap)
# slab          → kernel slab alocado para esse cgroup
# sock          → buffers de socket

# O OOM killer usa working_set = memory.current - inactive file pages
# Para ver o que realmente importa:
# anon + sock + ativo_file = pressão real de memória
```

---

## Referências

- **Brendan Gregg** — [Linux Performance](https://brendangregg.com/linuxperf.html), cgroups e namespaces no contexto de performance
- **Kernel docs** — [cgroups v2](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
- **Kernel docs** — [namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- **Linuxtips DK8S** — Day2: containers internals, `/mnt/c/Users/felip/Documents/Kubernets/DK8S`
- **runc source** — github.com/opencontainers/runc (libcontainer/cgroups/)
- **containerd source** — github.com/containerd/cgroups
- **Liz Rice** — "Container Security" (O'Reilly) — namespaces e cgroups do zero
- Rootless containers: [rootlesscontaine.rs](https://rootlesscontaine.rs/)
