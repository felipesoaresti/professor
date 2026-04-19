---
tags:
  - linux
  - oom-killer
  - memoria
  - kernel
  - containers
area: linux
tipo: conteudo
prerequisites:
  - "[[01-Linux/cgroups-namespaces]]"
next:
  - "[[11-Exercicios/oom-killer]]"
trilha: "[[00-Trilha/linux]]"
---

# OOM Killer

> Conteúdo: [[01-Linux/oom-killer]] | Exercícios: [[11-Exercicios/oom-killer]] | Trilha: [[00-Trilha/linux]]

O OOM Killer (Out-Of-Memory Killer) é o mecanismo do kernel que entra em ação quando o sistema fica sem memória disponível e não consegue mais satisfazer uma alocação. Em vez de deixar o kernel travar, ele escolhe um processo para matar. A escolha não é aleatória — existe um algoritmo de scoring que você pode influenciar.

---

## O que é e por que existe

Linux usa **overcommit** de memória por padrão: quando um processo faz `malloc(1GB)`, o kernel promete a memória mas não aloca páginas físicas imediatamente. As páginas só são alocadas quando o processo realmente escreve nelas (page fault → alocação real). Isso permite rodar mais cargas de trabalho do que a RAM física comportaria se todo mundo alocasse o máximo ao mesmo tempo.

O problema: quando vários processos tentam usar memória comprometida ao mesmo tempo, o sistema pode entrar em estado onde não há memória real suficiente para satisfazer as demandas. O kernel tenta liberar memória (page cache, swap), mas se não conseguir, o OOM killer é invocado.

**Sem o OOM killer:** o kernel travaria tentando alocar uma página que não existe.  
**Com o OOM killer:** um processo é sacrificado, o sistema sobrevive.

---

## Como funciona internamente

### O score de OOM

Cada processo tem um `oom_score` em `/proc/<PID>/oom_score` — um valor de 0 a 1000. O processo com maior score é o primeiro candidato a ser morto.

O algoritmo de scoring (função `oom_badness()` no kernel) considera:

1. **Tamanho do processo** (`mm->total_vm`) — quanto de memória virtual o processo usa. Processos grandes têm score maior.
2. **Filhos** — filhos contribuem com parte do seu uso para o score do pai
3. **Tempo de CPU consumido** — processos que rodaram por muito tempo recebem um pequeno desconto (o kernel reluta em matar trabalho já feito)
4. **`oom_score_adj`** — ajuste manual configurável por processo ou cgroup

```
score = tamanho_processo_em_páginas / total_ram_páginas * 1000 + oom_score_adj
```

> [!info] O score é calculado na hora do OOM
> O `oom_score` em `/proc/PID/oom_score` é uma estimativa que o kernel atualiza periodicamente, mas o score real é recalculado no momento do OOM event.

### `oom_score_adj` — o controle que você tem

O único parâmetro diretamente configurável é `oom_score_adj` em `/proc/<PID>/oom_score_adj`:

| Valor | Efeito |
|---|---|
| `-1000` | **Nunca matar** — processo imune ao OOM killer |
| `-999` a `-1` | Score reduzido — menos provável de ser morto |
| `0` | Padrão — sem ajuste |
| `1` a `999` | Score aumentado — mais provável de ser morto |
| `1000` | **Sempre matar primeiro** — alvo preferencial |

```bash
# Ver score atual de um processo
cat /proc/<PID>/oom_score
cat /proc/<PID>/oom_score_adj

# Proteger um processo crítico (requer root)
echo -1000 > /proc/<PID>/oom_score_adj

# Tornar um processo sacrificável primeiro
echo 1000 > /proc/<PID>/oom_score_adj
```

> [!warning] oom_score_adj vs oom_adj (legado)
> `/proc/PID/oom_adj` (valores -17 a 15) é a interface antiga, deprecated desde kernel 3.5. Use sempre `oom_score_adj`. Escrever em `oom_adj` ainda funciona mas converte internamente para `oom_score_adj`.

### Overcommit e suas configurações

```bash
# Ver política de overcommit atual
cat /proc/sys/vm/overcommit_memory
# 0 = heurística (padrão): permite overcommit razoável
# 1 = sempre permite (malloc nunca falha, OOM killer mais agressivo)
# 2 = nunca overcommit: malloc falha se não há memória real

cat /proc/sys/vm/overcommit_ratio
# Usado quando overcommit_memory=2: permite alocar até (RAM * ratio/100 + swap)

# Proporção de overcommit atual
cat /proc/sys/vm/committed_as   # total comprometido
cat /proc/meminfo | grep CommitLimit  # limite com policy atual
```

### O que acontece durante um OOM event

1. Um processo tenta alocar memória via `page_fault` ou `do_mmap`
2. O kernel não tem páginas livres → tenta liberar page cache (`kswapd`)
3. Tenta swap se configurado
4. Falha tudo → chama `out_of_memory()`
5. `oom_badness()` calcula score de todos os processos candidatos
6. Processo com maior score recebe `SIGKILL`
7. Kernel registra o evento em dmesg

### OOM do cgroup vs OOM do sistema

Com cgroups v2 e `memory.max` configurado (todo container Kubernetes com limits):

- O OOM killer é disparado **dentro do cgroup** quando o processo excede `memory.max`
- Apenas processos **dentro daquele cgroup** são candidatos
- O restante do sistema não é afetado
- Isso é o `OOMKilled` que você vê no `kubectl describe pod`

```
OOM do cgroup → mata processo dentro do container → container reinicia
OOM do sistema → mata qualquer processo no host → pode derrubar serviços críticos
```

---

## Na prática — comandos e observação

### Lendo o dmesg após um OOM

```bash
# Filtrar eventos OOM no dmesg
dmesg | grep -i "oom\|killed process\|out of memory"

# Exemplo de saída de um OOM event:
# [1234567.890] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),
#   cpuset=cri-containerd-abc123,mems_allowed=0,
#   oom_memcg=/kubepods.slice/.../cri-containerd-abc123.scope,
#   task_memcg=/kubepods.slice/.../cri-containerd-abc123.scope,
#   task=python3,pid=4567,uid=0
# [1234567.891] Memory cgroup out of memory: Killed process 4567 (python3)
#   total-vm:524288kB, anon-rss:102400kB, file-rss:8192kB, shmem-rss:0kB
#   oom_score_adj:974

# No systemd journal (mais estruturado):
journalctl -k | grep -i oom
journalctl -k --since "1 hour ago" | grep -i "killed\|oom"
```

### Identificar processos de alto risco

```bash
# Listar processos ordenados por oom_score (maiores riscos primeiro)
for pid in /proc/[0-9]*/oom_score; do
  score=$(cat "$pid" 2>/dev/null)
  adj=$(cat "${pid%oom_score}oom_score_adj" 2>/dev/null)
  name=$(cat "${pid%oom_score}comm" 2>/dev/null)
  echo "$score $adj $name"
done | sort -rn | head -20
```

```bash
# Ver oom_score_adj de todos os containers em execução (via cgroup)
for pid in $(crictl ps -q | xargs -I{} crictl inspect {} | grep '"pid"' | grep -o '[0-9]*'); do
  score=$(cat /proc/$pid/oom_score 2>/dev/null)
  adj=$(cat /proc/$pid/oom_score_adj 2>/dev/null)
  comm=$(cat /proc/$pid/comm 2>/dev/null)
  echo "PID=$pid score=$score adj=$adj cmd=$comm"
done
```

### Kubernetes e oom_score_adj

O kubelet configura `oom_score_adj` automaticamente baseado na QoS class do pod:

| QoS Class | `oom_score_adj` | Quando |
|---|---|---|
| **Guaranteed** | `-997` | requests == limits para CPU e memória |
| **Burstable** | `2` a `999` | limits > requests, ou só requests |
| **BestEffort** | `1000` | sem requests nem limits |

```bash
# Confirmar QoS de um pod
kubectl get pod <nome> -o jsonpath='{.status.qosClass}'

# Confirmar oom_score_adj do container no host
cat /proc/<PID>/oom_score_adj
# Guaranteed → deve ser -997
# BestEffort → deve ser 1000
```

> [!tip] Proteja workloads críticos com Guaranteed QoS
> Se um pod não pode ser OOM killed (banco de dados, cache crítico), configure `requests == limits` para CPU e memória. O kubelet vai setar `oom_score_adj=-997`, tornando o processo quase imune ao OOM killer do sistema.

### Configurar proteção de processo crítico (fora do K8s)

```bash
# Proteger o processo do containerd de OOM kills do sistema
PID=$(pgrep containerd)
echo -500 > /proc/${PID}/oom_score_adj

# Persisting via systemd (o processo reinicia e perde o adj)
# Em /etc/systemd/system/containerd.service.d/oom.conf:
[Service]
OOMScoreAdjust=-500

systemctl daemon-reload && systemctl restart containerd
```

---

## Casos de uso e boas práticas

### Quando o sistema começa a ter pressão de memória — sinais precoces

```bash
# Verificar se o kernel está em modo de pressão
cat /proc/meminfo | grep -E "MemAvailable|MemFree|SwapFree|Dirty|Writeback"

# Verificar kswapd ativo (kernel tentando liberar memória)
cat /proc/vmstat | grep pgscand   # páginas scanned pelo scanner de memória

# Verificar se há processos em OOM wait
dmesg | tail -50 | grep -i oom

# Verificar uso de swap
vmstat 1 5  # coluna si/so = swap in/out
```

### Configurar `vm.panic_on_oom`

Em servidores críticos onde é melhor reiniciar do que ficar em estado degradado:

```bash
# Opção: panicar em vez de rodar o OOM killer
sysctl vm.panic_on_oom=1
# kernel.panic=30 → reinicia após 30s de kernel panic
```

> [!warning] Não use `panic_on_oom` em nodes Kubernetes
> O kubelet e o cgroup OOM já tratam OOM de containers de forma isolada. Panic do kernel derrubaria o node inteiro por uma falha de um único pod.

---

## Troubleshooting — cenários reais de produção

### Cenário 1: Pod com OOMKilled repetido

```bash
# Confirmar OOMKilled
kubectl describe pod <nome> | grep -A5 "Last State"
# Last State: Terminated
#   Reason: OOMKilled

# Ver limite de memória configurado
kubectl get pod <nome> -o jsonpath='{.spec.containers[0].resources}'

# No node: ver memory.events do cgroup
crictl inspect <container-id> | grep '"pid"'
CGROUP=$(cat /proc/<PID>/cgroup | cut -d: -f3)
cat /sys/fs/cgroup${CGROUP}/memory.events
# oom: 3   ← quantas vezes chegou ao limite
# oom_kill: 3

# Ver decomposição de memória para entender a causa
cat /sys/fs/cgroup${CGROUP}/memory.stat | grep -E "^(anon|file|slab|sock) "
```

**Ações:**
- Se `anon` alto → aplicação tem memory leak ou precisa de mais memória real → aumentar limits
- Se `file` alto → page cache recuperável → o pod pode não precisar de mais limits, só de um `memory.high`

### Cenário 2: Node com OOM kill de processos do sistema (não containers)

```bash
# Ver qual processo foi morto
dmesg | grep "Killed process" | tail -5

# Verificar se foi OOM de cgroup ou sistema
dmesg | grep -A2 "Out of memory"
# "constraint=CONSTRAINT_MEMCG" → OOM de cgroup (container)
# "constraint=CONSTRAINT_NONE"  → OOM do sistema (grave)

# Se OOM do sistema: identificar qual cgroup está consumindo mais
for d in /sys/fs/cgroup/kubepods.slice/**/*.scope/memory.current; do
  echo "$(cat $d) $d"
done | sort -rn | head -10
```

### Cenário 3: Processo crítico foi morto pelo OOM killer

```bash
# Identificar o processo morto
dmesg | grep "Killed process"

# Verificar qual era o oom_score_adj (estava protegido?)
dmesg | grep -B5 "Killed process" | grep oom_score_adj

# Se não estava protegido:
# 1. Adicionar OOMScoreAdjust=-1000 na unit do systemd
# 2. Para K8s: mudar para Guaranteed QoS
# 3. Adicionar memory requests/limits adequados para o node não ficar sem memória
```

---

## Nível avançado — kernel internals, edge cases

### A função `oom_badness()` no kernel

```c
// Simplificado de mm/oom_kill.c
long oom_badness(struct task_struct *p, unsigned long totalpages) {
    long points;
    long adj;

    adj = (long)p->signal->oom_score_adj;
    if (adj == OOM_SCORE_ADJ_MIN)  // -1000
        return LONG_MIN;  // nunca matar

    points = get_mm_rss(p->mm);  // RSS em páginas
    points += get_mm_counter(p->mm, MM_SWAPENTS);  // swap usado
    points += mm_pgtables_bytes(p->mm) / PAGE_SIZE;  // page tables

    points += adj * totalpages / 1000;  // aplica oom_score_adj

    return points;
}
```

O kernel mata o processo com maior `points`. Se dois processos têm o mesmo score, mata o mais novo (menor uptime).

### OOM killer e threads

O OOM killer mata o **thread group leader** e todos os threads do grupo. Um processo multi-threaded (ex: Java com 200 threads) é tratado como uma unidade — score calculado para o conjunto, kill derruba todos os threads.

```bash
# Ver threads de um processo
ls /proc/<PID>/task/
cat /proc/<PID>/status | grep Threads
```

### cgroup OOM e `memory.oom.group`

No cgroup v2, é possível configurar que quando um processo do cgroup for OOM-killed, **todos os processos do cgroup sejam mortos juntos**:

```bash
# Matar todo o cgroup quando um processo excede o limite (em vez de só o processo)
echo 1 > /sys/fs/cgroup/<grupo>/memory.oom.group
```

Isso é exatamente o que o containerd configura para containers — quando o container excede o limite, o cgroup inteiro é limpo (todos os processos do container são mortos, não só o que causou o OOM).

### `vm.overcommit_memory=2` em produção — banco de dados

Para bancos de dados (PostgreSQL, etc.), é comum usar `overcommit_memory=2` para que o `malloc` falhe com erro ao invés de deixar o OOM killer matar o processo no pior momento possível (durante uma transação):

```bash
# PostgreSQL prefere falhar no malloc do que ser OOM killed
sysctl vm.overcommit_memory=2
sysctl vm.overcommit_ratio=50  # RAM * 50% + swap = limite total

# Verificar se o sistema está abaixo do commit limit
grep -E "CommitLimit|Committed_AS" /proc/meminfo
# Committed_AS deve ser < CommitLimit
```

### Detectar memory leak de container antes do OOM

```bash
# Monitorar crescimento de memória de um cgroup ao longo do tempo
CGROUP=$(cat /proc/<PID>/cgroup | cut -d: -f3)
while true; do
  echo "$(date): $(cat /sys/fs/cgroup${CGROUP}/memory.current) bytes"
  sleep 10
done
# Se crescimento linear e sem platô → memory leak
```

---

## Referências

- **Kernel source** — `mm/oom_kill.c`, `mm/page_alloc.c`
- **Kernel docs** — [overcommit accounting](https://www.kernel.org/doc/html/latest/vm/overcommit-accounting.html)
- **Brendan Gregg** — "Systems Performance" cap. 7 (Memory)
- **Kubernetes docs** — [Configure Out of Resource Handling](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/)
- `man 5 proc` — `/proc/PID/oom_score`, `/proc/PID/oom_score_adj`
- **lwn.net** — [Taming the OOM killer](https://lwn.net/Articles/317814/)
