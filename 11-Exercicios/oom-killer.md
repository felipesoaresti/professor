---
tags:
  - exercicios
  - linux
  - oom-killer
  - memoria
tipo: exercicios
area: linux
conteudo: "[[01-Linux/oom-killer]]"
trilha: "[[00-Trilha/linux]]"
---

# Exercícios — OOM Killer

> Conteúdo: [[01-Linux/oom-killer]] | Trilha: [[00-Trilha/linux]]

**Infraestrutura:** k8s-master (192.168.3.30), worker1 (192.168.3.31), worker2 (192.168.3.32)
**Atenção:** Não tocar em `controle-gastos`, `promobot`, `databases/postgres`.

---

## Exercício 1: Lendo oom_score de todos os processos

**Contexto:** Você está fazendo um hardening de um node Kubernetes e quer entender quais processos têm maior risco de serem mortos por um OOM event.

**Missão:** No k8s-master, liste os 15 processos com maior `oom_score` e o `oom_score_adj` correspondente. Identifique qual é o kubelet e qual é o containerd.

**Requisitos:**
- [ ] Ler `oom_score` de todos os processos em `/proc` sem usar ferramentas externas além de bash/awk
- [ ] Listar os 15 processos com maior score, incluindo nome e `oom_score_adj`
- [ ] Identificar o score do kubelet e do containerd
- [ ] Identificar qual processo tem `oom_score_adj=-997` e por que esse valor específico
- [ ] Explicar por que o processo com maior score seria o primeiro candidato a morrer

**Verificação:**
```bash
for pid in /proc/[0-9]*/oom_score; do
  score=$(cat "$pid" 2>/dev/null)
  [ -n "$score" ] && echo "$score $pid"
done | sort -rn | head -15
```

---

## Exercício 2: Inspecionar QoS e oom_score_adj em pods Kubernetes

**Contexto:** O time de SRE quer garantir que os pods críticos têm proteção adequada contra OOM kills.

**Missão:** Para pelo menos 3 pods em execução com QoS classes diferentes, confirme a relação entre QoS class e `oom_score_adj` diretamente no cgroup/proc.

**Requisitos:**
- [ ] Identificar pods com QoS `Guaranteed`, `Burstable` e `BestEffort` no cluster
- [ ] Para cada pod, obter o PID do container principal
- [ ] Ler `oom_score_adj` de cada PID via `/proc`
- [ ] Confirmar que Guaranteed → `-997`, BestEffort → `1000`
- [ ] Criar um pod Guaranteed (requests == limits) e confirmar o `oom_score_adj` resultante
- [ ] Remover o pod criado

**Verificação:**
```bash
kubectl get pod <nome> -o jsonpath='{.status.qosClass}'
cat /proc/<PID>/oom_score_adj
# Deve bater com a tabela: Guaranteed=-997, BestEffort=1000
```

---

## Exercício 3: Provocar e observar um OOM kill de cgroup

**Contexto:** A equipe quer entender exatamente o que acontece quando um pod excede o memory limit. Você vai criar um pod com limite baixo, provocar o OOM, e analisar todas as evidências.

**Missão:** Crie um pod com `memory.limits=50Mi`, force um consumo de memória maior que o limite, e colete todas as evidências do OOM kill.

**Requisitos:**
- [ ] Criar pod com `limits.memory=50Mi` e `requests.memory=10Mi` no namespace `default`
- [ ] Dentro do pod, executar um comando que aloque mais de 50Mi (ex: `dd`, `python -c "x='a'*100000000"`)
- [ ] Observar o pod ser reiniciado com `OOMKilled`
- [ ] No node, ler `dmesg` e identificar a linha exata do OOM kill
- [ ] Ler `memory.events` do cgroup do container e registrar os valores de `oom` e `oom_kill`
- [ ] Confirmar que foi um OOM de cgroup (`constraint=CONSTRAINT_MEMCG`) e não do sistema
- [ ] Deletar o pod ao final

**Verificação:**
```bash
kubectl describe pod <nome> | grep -A5 "Last State"
# Reason: OOMKilled

dmesg | grep "Memory cgroup out of memory"
```

---

## Exercício 4: Proteger processo crítico do OOM killer

**Contexto:** O containerd e o kubelet são críticos para a saúde do node. Se forem mortos por OOM, o node interage mal com o cluster. Você precisa garantir proteção permanente.

**Missão:** Configure proteção de OOM para o containerd e o kubelet via systemd drop-in, de forma que sobreviva a reinicializações dos serviços.

**Requisitos:**
- [ ] Verificar o `oom_score_adj` atual do containerd e do kubelet antes da mudança
- [ ] Criar drop-in systemd em `/etc/systemd/system/containerd.service.d/oom.conf` com `OOMScoreAdjust=-500`
- [ ] Fazer o mesmo para o kubelet
- [ ] Recarregar systemd e reiniciar os serviços
- [ ] Confirmar que o `oom_score_adj` mudou no `/proc` após o restart
- [ ] Verificar que o cluster ainda está saudável após os restarts

**Verificação:**
```bash
systemctl show containerd | grep OOMScore
cat /proc/$(pgrep containerd)/oom_score_adj
# Deve ser -500
kubectl get nodes  # cluster ainda OK
```

---

## Exercício 5: Analisar um OOM kill histórico no dmesg

**Contexto:** Você assumiu o on-call e há um alerta de pod restartando repetidamente desde a madrugada. Você precisa fazer o post-mortem do OOM kill.

**Missão:** No k8s-master, analise o dmesg e o journal para encontrar todos os OOM events das últimas 24h e reconstruir o que aconteceu.

**Requisitos:**
- [ ] Buscar no dmesg todos os OOM kills registrados
- [ ] Para cada evento: identificar o processo morto, o PID, o oom_score_adj, e o cgroup
- [ ] Determinar se foi OOM de cgroup (container) ou OOM do sistema
- [ ] Usar `journalctl -k` para cruzar com timestamps e identificar qual pod foi afetado
- [ ] Correlacionar com `kubectl get events --field-selector reason=OOMKilling -A`
- [ ] Registrar a linha de evidência completa: dmesg → journal → kubectl events

**Verificação:**
```bash
dmesg | grep -E "oom_kill|Killed process|Out of memory"
journalctl -k --since "24 hours ago" | grep -i oom
kubectl get events -A --field-selector reason=OOMKilling
```

---

## Exercício 6: Overcommit e CommitLimit (avançado)

**Contexto:** Você está dimensionando um node Kubernetes e precisa entender os limites reais de alocação de memória antes que o OOM killer seja invocado.

**Missão:** No k8s-master, analise a configuração de overcommit atual, calcule o CommitLimit real, e determine quantas alocações ainda cabem antes do sistema entrar em risco.

**Requisitos:**
- [ ] Ler a política atual de overcommit (`/proc/sys/vm/overcommit_memory`)
- [ ] Ler `CommitLimit` e `Committed_AS` do `/proc/meminfo`
- [ ] Calcular a margem disponível antes de atingir o commit limit
- [ ] Calcular qual seria o CommitLimit com `overcommit_memory=2` e `overcommit_ratio=80`
- [ ] Identificar quais são os maiores consumidores de memória comprometida no momento
- [ ] Responder: o node atual está em risco de OOM com os pods atuais?

**Verificação:**
```bash
grep -E "CommitLimit|Committed_AS|MemTotal|SwapTotal" /proc/meminfo
# CommitLimit deve ser > Committed_AS (margem positiva = sem risco imediato)
sysctl vm.overcommit_memory vm.overcommit_ratio
```

---

## Exercício 7: Simular decisão do OOM killer (Staff-level)

**Contexto:** Você precisa prever qual processo o OOM killer vai matar em um cenário de produção antes que o evento aconteça.

**Missão:** Sem provocar um OOM real, simule o algoritmo de score do OOM killer para os processos em execução no node e preveja quem seria sacrificado.

**Requisitos:**
- [ ] Para os 20 processos com maior RSS (`/proc/PID/status` → `VmRSS`), calcular o score estimado
- [ ] Fórmula: `score = (VmRSS_em_kB / MemTotal_em_kB) * 1000 + oom_score_adj`
- [ ] Listar os 5 candidatos mais prováveis para um OOM kill do sistema (não de cgroup)
- [ ] Verificar se algum processo crítico (kubelet, containerd, sshd) está entre os candidatos
- [ ] Se sim, propor e aplicar mitigação com `oom_score_adj`
- [ ] Recalcular o ranking após a mitigação

**Verificação:**
```bash
# Script de cálculo do score estimado
MEMTOTAL=$(grep MemTotal /proc/meminfo | awk '{print $2}')
for pid in /proc/[0-9]*/status; do
  rss=$(grep VmRSS "$pid" 2>/dev/null | awk '{print $2}')
  adj=$(cat "${pid%status}oom_score_adj" 2>/dev/null)
  name=$(grep ^Name "$pid" 2>/dev/null | awk '{print $2}')
  [ -n "$rss" ] && [ -n "$adj" ] && \
    echo "$(( rss * 1000 / MEMTOTAL + adj )) $name (adj=$adj)"
done | sort -rn | head -20
```

---

> [!tip] Ordem recomendada
> Exercícios 1 e 2 são observatórios — sem risco, faça primeiro.
> Exercício 3 provoca OOM real em pod descartável — confirme que tem namespace `default` limpo.
> Exercício 4 mexe em serviços críticos — confirme acesso SSH alternativo antes de reiniciar containerd.
> Exercícios 5, 6, 7 são investigativos e analíticos — podem ser feitos em qualquer ordem.
