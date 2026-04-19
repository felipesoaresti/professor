---
tags:
  - exercicios
  - linux
  - processos
  - scheduling
  - cfs
tipo: exercicios
area: linux
conteudo: "[[01-Linux/processos-scheduling]]"
trilha: "[[00-Trilha/linux]]"
---

# Exercícios — Processos e Scheduling

> Conteúdo: [[01-Linux/processos-scheduling]] | Trilha: [[00-Trilha/linux]]

**Infraestrutura:** k8s-master (192.168.3.30), worker1 (192.168.3.31), worker2 (192.168.3.32)
**Atenção:** Não tocar em `controle-gastos`, `promobot`, `databases/postgres`.

---

## Exercício 1: Estados de processo e o que eles significam

**Contexto:** Você recebe um alerta: load average em 8.0 num node com 4 CPUs, mas o Grafana mostra apenas 20% de CPU utilizado. O time acha que é bug do Prometheus. Você sabe que não é.

**Missão:** No k8s-master, identifique todos os processos em estado D e S, entenda por que o load está alto apesar das CPUs livres, e localize onde cada processo D está travado no kernel.

**Requisitos:**
- [ ] Listar todos os processos em D-state via `ps`
- [ ] Para cada processo em D, ler `/proc/<PID>/wchan` (função do kernel onde está dormindo)
- [ ] Para cada processo em D, ler `/proc/<PID>/stack` (stack completo do kernel)
- [ ] Ler o load average atual e confirmar que D-state contribui para ele
- [ ] Explicar por que `kill -9 <PID>` não funciona em processos D
- [ ] Identificar se algum D-state é de I/O de disco ou de NFS/rede

**Verificação:**
```bash
ps -eo pid,stat,comm,wchan | awk '$2 ~ /^D/'
# Se nenhum processo D agora, forçar: dd if=/dev/sda of=/dev/null bs=1M &
# e observar o estado enquanto roda
```

---

## Exercício 2: Inspecionando processos via /proc sem ferramentas externas

**Contexto:** Você está em um container de debug minimalista sem `ps`, `top`, `htop` ou qualquer ferramenta. Só tem bash e acesso ao `/proc`.

**Missão:** Usando apenas leitura de `/proc`, descubra os 5 processos com maior RSS no sistema, seus estados, PPIDs e tempo total de CPU consumido.

**Requisitos:**
- [ ] Iterar por todos os PIDs em `/proc` usando apenas bash e `cat`
- [ ] Para cada processo, ler `VmRSS` de `/proc/<PID>/status`
- [ ] Listar os 5 com maior RSS com nome, PID, PPID e estado
- [ ] Para o processo de maior RSS, ler `/proc/<PID>/stat` e extrair o tempo de CPU (campos `utime` e `stime`)
- [ ] Converter de jiffies para segundos (dividir por `getconf CLK_TCK`)
- [ ] Listar os file descriptors abertos pelo processo de maior RSS

**Verificação:**
```bash
for pid in /proc/[0-9]*/status; do
  rss=$(grep VmRSS "$pid" 2>/dev/null | awk '{print $2}')
  name=$(grep ^Name "$pid" 2>/dev/null | awk '{print $2}')
  [ -n "$rss" ] && echo "$rss $name"
done | sort -rn | head -5
```

---

## Exercício 3: Analisando context switches e scheduling

**Contexto:** Um pod de aplicação está com latência inconsistente — às vezes responde em 5ms, às vezes em 200ms. O time suspeita que o processo está sendo preemptado pelo scheduler com frequência.

**Missão:** Para um processo em execução (pode ser qualquer pod com atividade), analise o padrão de context switches e determine se é um processo CPU-bound (muitos nonvoluntary) ou I/O-bound (muitos voluntary).

**Requisitos:**
- [ ] Escolher um processo em execução (container ou sistema)
- [ ] Ler context switches via `/proc/<PID>/status` em dois momentos com 10s de intervalo
- [ ] Calcular taxa de context switches por segundo (voluntary e nonvoluntary)
- [ ] Ler `/proc/<PID>/sched` e registrar `nr_switches`, `wait_sum`, `se_sum_exec_runtime`
- [ ] Calcular o percentual de tempo que o processo ficou esperando na runqueue: `wait_sum / (wait_sum + se_sum_exec_runtime)`
- [ ] Classificar: CPU-bound, I/O-bound, ou scheduling-starved?

**Verificação:**
```bash
# Snapshot 1
grep ctxt /proc/<PID>/status

sleep 10

# Snapshot 2
grep ctxt /proc/<PID>/status

# Delta = switches no intervalo / 10 = switches/segundo
```

---

## Exercício 4: Load average — diagnóstico real

**Contexto:** Em um node de produção, o load average está em 6.0 com 4 CPUs. Você precisa diagnosticar a causa raiz em menos de 5 minutos.

**Missão:** No k8s-master, execute um diagnóstico completo de load average seguindo o método correto (não apenas olhar CPU%).

**Requisitos:**
- [ ] Ler load average atual e número de CPUs (`nproc`)
- [ ] Calcular se o load está acima da capacidade de CPU (`load / nproc > 1.0`)
- [ ] Verificar CPU utilization real com `mpstat` ou lendo `/proc/stat`
- [ ] Se CPUs idle mas load alto → identificar processos em D-state
- [ ] Se CPUs saturadas → identificar top consumers com `ps` + `%cpu`
- [ ] Para simular load por D-state: `stress --io 4 --timeout 30s &` e observar load subir sem CPU saturar
- [ ] Usar `vmstat 1 10` e interpretar todas as colunas relevantes (r, b, wa, st)

**Verificação:**
```bash
uptime
mpstat 1 5
vmstat 1 5
ps -eo pid,stat,comm | awk '$2 ~ /^D/' | wc -l
# Correlacionar: load = procs_R + procs_D
```

---

## Exercício 5: Zombie processes — identificar e resolver

**Contexto:** Um desenvolvedor reportou que um processo pai está criando filhos que nunca são coletados. Os zombies estão acumulando e o time teme esgotar a tabela de PIDs.

**Missão:** Crie um zombie process propositalmente, observe-o no sistema, identifique o pai, e limpe sem matar o processo pai com `kill -9`.

**Requisitos:**
- [ ] Criar um zombie process com este one-liner Python:
  ```bash
  python3 -c "
  import os, time
  pid = os.fork()
  if pid > 0:
      print(f'Pai: {os.getpid()}, Filho: {pid}')
      time.sleep(3600)  # pai não chama wait()
  else:
      os._exit(0)  # filho termina, vira zombie
  " &
  ```
- [ ] Confirmar o zombie com `ps aux | grep Z`
- [ ] Ler `/proc/<ZOMBIE-PID>/status` e registrar os campos disponíveis (zombie tem status limitado)
- [ ] Identificar o PPID do zombie
- [ ] Enviar `SIGCHLD` ao pai para forçar o `wait()` sem matá-lo: `kill -CHLD <PPID>`
- [ ] Confirmar que o zombie desapareceu
- [ ] Limpar o processo pai ao final

**Verificação:**
```bash
ps -eo pid,ppid,stat,comm | grep " Z "
kill -CHLD <PPID>
ps -eo pid,ppid,stat,comm | grep " Z "  # zombie deve ter sumido
```

---

## Exercício 6: Nice e CPU affinity

**Contexto:** Você tem um job de análise de logs que é CPU-intensivo e está impactando a latência de outros serviços no mesmo node. Você precisa despriorizá-lo sem matá-lo.

**Missão:** Execute um processo CPU-intensivo com nice baixo, observe o impacto no scheduling, e depois aplique nice alto e CPU affinity para isolá-lo.

**Requisitos:**
- [ ] Iniciar um processo CPU-intensivo em background: `dd if=/dev/zero of=/dev/null &`
- [ ] Observar com `ps -eo pid,ni,pcpu,comm` que está consumindo CPU com nice 0
- [ ] Aplicar `renice +19` no processo
- [ ] Confirmar via `/proc/<PID>/status` que `nice` mudou
- [ ] Verificar que o CPU usage do processo caiu quando há outros processos concorrendo
- [ ] Aplicar CPU affinity para restringir o processo à CPU 0 apenas (`taskset`)
- [ ] Confirmar a affinity via `/proc/<PID>/status` campo `Cpus_allowed_list`
- [ ] Encerrar o processo ao final

**Verificação:**
```bash
ps -eo pid,ni,pcpu,comm | grep dd
cat /proc/<PID>/status | grep -E "^(Cpus_allowed|voluntary)"
taskset -p <PID>
```

---

## Exercício 7: Rastreando fork e exec com strace (avançado)

**Contexto:** Uma aplicação está invocando processos filhos de forma inesperada e você precisa descobrir exatamente quais comandos são executados, sem ter acesso ao código-fonte.

**Missão:** Use `strace` para rastrear todos os `fork()`, `clone()` e `execve()` de um processo e seus filhos, construindo a árvore de execução.

**Requisitos:**
- [ ] Escolher um processo que cria filhos periodicamente (ex: `kubelet`, `containerd`, ou `bash -c "while true; do ls; sleep 5; done"`)
- [ ] Rastrear com `strace -f -e trace=clone,fork,vfork,execve -p <PID>` por 30 segundos
- [ ] Registrar cada `execve` capturado: qual binário foi executado e com quais argumentos
- [ ] Identificar quantos processos filhos foram criados no período
- [ ] Usar `strace -c` para obter o sumário de syscalls e identificar as mais frequentes
- [ ] Calcular o overhead aproximado do strace comparando CPU usage antes e durante o trace

**Verificação:**
```bash
strace -f -e trace=execve -p <PID> -o /tmp/trace.log &
sleep 30
kill %1
grep execve /tmp/trace.log | wc -l
grep execve /tmp/trace.log | head -20
```

---

## Exercício 8: Correlacionando cpu.weight do cgroup com CFS (Staff-level)

**Contexto:** O time questiona se o `requests.cpu` do Kubernetes realmente afeta o scheduling ou é apenas um valor para o scheduler de pods. Você vai provar empiricamente.

**Missão:** Crie dois pods com `requests.cpu` diferentes, force contenção de CPU no node, e confirme que o CFS distribui CPU proporcionalmente aos pesos dos cgroups.

**Requisitos:**
- [ ] Criar pod-A com `requests.cpu=200m` e pod-B com `requests.cpu=600m` (sem limits) no namespace `default`
- [ ] Confirmar os `cpu.weight` nos cgroups dos dois containers (ratio deve ser ~1:3)
- [ ] Iniciar carga de CPU em ambos os pods simultaneamente: `dd if=/dev/zero of=/dev/null`
- [ ] Observar por 60 segundos com `kubectl top pod` — pod-B deve receber ~3x mais CPU que pod-A
- [ ] Confirmar no cgroup lendo `cpu.stat` → `usage_usec` de cada container
- [ ] Calcular a proporção real observada vs. a proporção teórica pelos pesos
- [ ] Deletar os pods ao final

**Verificação:**
```bash
CGROUP_A=$(cat /proc/<PID-A>/cgroup | cut -d: -f3)
CGROUP_B=$(cat /proc/<PID-B>/cgroup | cut -d: -f3)
cat /sys/fs/cgroup${CGROUP_A}/cpu.weight
cat /sys/fs/cgroup${CGROUP_B}/cpu.weight
# Ratio deve ser aproximadamente 204:614 (proporcional aos millicores)
```

---

> [!tip] Ordem recomendada
> Exercícios 1 e 2 são observatórios — sem risco, comece por eles.
> Exercício 3 requer processo com atividade real — escolha um pod com tráfego.
> Exercício 4 usa `stress` para simular carga — instalar com `apt install stress` se necessário.
> Exercícios 5 e 6 manipulam processos locais — sem risco para o cluster.
> Exercícios 7 e 8 são avançados: strace tem overhead real, use em processo não crítico.
