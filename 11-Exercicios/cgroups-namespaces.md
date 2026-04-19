---
tags:
  - exercicios
  - linux
  - cgroups
  - namespaces
  - containers
tipo: exercicios
area: linux
conteudo: "[[01-Linux/cgroups-namespaces]]"
trilha: "[[00-Trilha/linux]]"
---

# Exercícios — Cgroups e Namespaces

> Conteúdo: [[01-Linux/cgroups-namespaces]] | Trilha: [[00-Trilha/linux]]

**Infraestrutura:** k8s-master (192.168.3.30), worker1 (192.168.3.31), worker2 (192.168.3.32)
**Atenção:** Não tocar em `controle-gastos`, `promobot`, `databases/postgres`.

---

## Exercício 1: Mapeando namespaces do sistema

**Contexto:** Você assumiu o on-call de um cluster Kubernetes e precisa entender a estrutura de namespaces antes de qualquer troubleshooting.

**Missão:** No k8s-master, liste todos os namespaces do sistema e identifique quantos net namespaces existem. Encontre o PID de um container em execução qualquer e liste os namespaces aos quais ele pertence via `/proc`.

**Requisitos:**
- [ ] Usar `lsns` para listar namespaces, filtrando por tipo `net`
- [ ] Identificar um container em execução com `crictl ps`
- [ ] Obter o PID do container no host via `crictl inspect`
- [ ] Listar os namespaces do processo via `/proc/<PID>/ns/` e observar os inodes
- [ ] Comparar o inode do net namespace do container com o do processo init (PID 1)

**Verificação:**
```bash
# O inode do net ns do container deve ser diferente do PID 1
ls -lai /proc/1/ns/net
ls -lai /proc/<PID-container>/ns/net
```

---

## Exercício 2: Entrando em namespaces sem kubectl exec

**Contexto:** O time reporta que um pod está com problemas de rede mas a imagem não tem shell (`distroless`). `kubectl exec` falha. Você precisa inspecionar a stack de rede do container diretamente do node.

**Missão:** Usando `nsenter`, inspecione a rede de um container em execução no worker1 sem usar `kubectl exec` ou `docker exec`.

**Requisitos:**
- [ ] Identificar um pod em execução no worker1
- [ ] Obter o PID do container principal no host
- [ ] Listar interfaces de rede dentro do namespace de rede do container via `nsenter`
- [ ] Listar sockets TCP em uso dentro do container via `nsenter`
- [ ] Mostrar a tabela de rotas dentro do container via `nsenter`
- [ ] Confirmar que a interface `eth0` dentro do container tem IP diferente do host

**Verificação:**
```bash
nsenter -t <PID> --net ip addr
# eth0 deve aparecer com IP do pod (ex: 10.x.x.x), não o IP do node
```

---

## Exercício 3: Criando um namespace isolado manualmente

**Contexto:** Você precisa testar um serviço de forma isolada sem criar um container completo — apenas com os primitivos do kernel.

**Missão:** Usando `unshare`, crie um ambiente com PID namespace isolado. Dentro desse ambiente, observe que você é o PID 1 e que não vê os processos do host.

**Requisitos:**
- [ ] Executar `unshare` com PID namespace e proc montado
- [ ] Dentro do ambiente, executar `ps aux` e confirmar que apenas seus processos aparecem
- [ ] Confirmar que você é PID 1 dentro do namespace
- [ ] Verificar do host (em outro terminal) qual é o PID real do processo usando `/proc/<PID>/status`
- [ ] Observar o campo `NSpid` no status — dois valores: PID no host e PID no namespace

**Verificação:**
```bash
# Dentro do unshare:
echo $$          # deve retornar 1

# No host, em outro terminal:
grep NSpid /proc/<PID-real>/status
# NSpid: <PID-host>  1
```

---

## Exercício 4: Lendo cgroups de containers em execução

**Contexto:** O time de plataforma quer saber exatamente quais limites de CPU e memória estão configurados para um pod específico — sem usar `kubectl describe` ou a API do Kubernetes.

**Missão:** Para um pod qualquer em execução, navegue pelo `/sys/fs/cgroup` e leia os limites e uso atual diretamente dos arquivos do kernel.

**Requisitos:**
- [ ] Obter PID de um container em execução
- [ ] Encontrar o caminho do cgroup via `/proc/<PID>/cgroup`
- [ ] Ler o limite de CPU (`cpu.max`) e interpretar o valor (quota/period)
- [ ] Ler o limite de memória (`memory.max`)
- [ ] Ler o uso atual de memória (`memory.current`)
- [ ] Ler `cpu.stat` e identificar os campos de throttling
- [ ] Comparar os limites encontrados com o que `kubectl describe pod` mostra

**Verificação:**
```bash
CGROUP=$(cat /proc/<PID>/cgroup | cut -d: -f3)
cat /sys/fs/cgroup${CGROUP}/cpu.max
cat /sys/fs/cgroup${CGROUP}/memory.max
cat /sys/fs/cgroup${CGROUP}/cpu.stat
```

---

## Exercício 5: Detectando CPU throttling

**Contexto:** Um pod de aplicação está com latência alta no p99 mas o `kubectl top pod` mostra 30% de CPU — bem abaixo do limite. O time suspeita de throttling mas não sabe como confirmar.

**Missão:** Implante um pod com CPU limits baixos, gere carga nele, e confirme o throttling lendo diretamente o cgroup.

**Requisitos:**
- [ ] Criar um pod com `limits.cpu: 100m` e `requests.cpu: 50m` em namespace `default`
- [ ] Gerar carga de CPU dentro do pod (ex: `dd if=/dev/zero of=/dev/null` ou loop infinito)
- [ ] No node onde o pod está, localizar o cgroup do container
- [ ] Monitorar `cpu.stat` em tempo real e observar `nr_throttled` aumentando
- [ ] Calcular a porcentagem de throttling: `throttled_usec / (nr_periods * period_duration)`
- [ ] Remover o pod ao final

**Verificação:**
```bash
watch -n1 "cat /sys/fs/cgroup\$(cat /proc/<PID>/cgroup | cut -d: -f3)/cpu.stat | grep -E 'throttled|nr_periods'"
# nr_throttled deve aumentar enquanto há carga
```

---

## Exercício 6: Criando cgroup manualmente e confinando um processo

**Contexto:** Você precisa testar o comportamento de uma aplicação quando atingir o limite de memória, mas sem criar um container completo.

**Missão:** Crie um cgroup v2 manualmente, configure um limite de 50MB de memória, e execute um processo dentro desse cgroup. Observe o que acontece quando o processo tenta alocar mais memória que o limite.

**Requisitos:**
- [ ] Verificar se o controller `memory` está habilitado no cgroup root
- [ ] Criar um cgroup filho em `/sys/fs/cgroup/teste-limite/`
- [ ] Configurar `memory.max` para 50MB (em bytes)
- [ ] Mover um processo para o cgroup escrevendo o PID em `cgroup.procs`
- [ ] Verificar que o processo está no cgroup correto via `/proc/<PID>/cgroup`
- [ ] Tentar alocar memória além do limite e observar o comportamento
- [ ] Verificar `memory.events` para confirmar o OOM kill
- [ ] Remover o cgroup ao final (`rmdir` — só funciona se não houver processos)

**Verificação:**
```bash
cat /sys/fs/cgroup/teste-limite/memory.events
# oom_kill deve ser > 0 após o processo tentar exceder o limite
```

---

## Exercício 7: Investigar namespace leak (avançado)

**Contexto:** Após um incidente, o time notou que o número de net namespaces no sistema continua crescendo mesmo após pods serem deletados. Você precisa identificar qual processo está segurando os namespaces.

**Missão:** Simule um leak mantendo uma referência a um net namespace aberta, identifique o "vazamento" usando `lsns` e `/proc`, e corrija-o.

**Requisitos:**
- [ ] Criar um net namespace com `ip netns add ns-teste`
- [ ] Abrir uma referência ao namespace em background (ex: `ip netns exec ns-teste sleep 3600 &`)
- [ ] Deletar o namespace com `ip netns delete ns-teste`
- [ ] Confirmar com `lsns -t net` que o namespace ainda existe (referência mantida pelo sleep)
- [ ] Identificar o PID do processo que segura o namespace pelo inode
- [ ] Matar o processo e confirmar que o namespace desapareceu
- [ ] Entender por que `ip netns delete` não eliminou o namespace imediatamente

**Verificação:**
```bash
# Após o delete, o namespace ainda deve aparecer
lsns -t net | grep <inode>

# Após matar o processo:
lsns -t net   # namespace não deve mais aparecer
```

---

## Exercício 8: Análise de memória de container (Staff-level)

**Contexto:** Um pod está próximo do limits de memória mas o time não sabe se a memória é recuperável (cache) ou se é pressão real. Reiniciar o pod vai resolver ou o problema vai voltar imediatamente?

**Missão:** Para um pod em execução, decomponha o uso de memória lendo `memory.stat` e identifique quanto é anônimo (pressão real) vs page cache (recuperável).

**Requisitos:**
- [ ] Escolher um pod com alguma atividade de I/O (ex: pod que lê/escreve arquivos)
- [ ] Ler `memory.stat` completo do cgroup
- [ ] Identificar e registrar os valores de: `anon`, `file`, `slab`, `sock`
- [ ] Calcular a "pressão real" de memória: `anon + sock`
- [ ] Calcular o que é recuperável: `file` (page cache)
- [ ] Comparar o total (`memory.current`) com a decomposição
- [ ] Concluir: se o pod fosse reiniciado, o uso de memória imediatamente voltaria ao mesmo nível?

**Verificação:**
```bash
CGROUP=$(cat /proc/<PID>/cgroup | cut -d: -f3)
cat /sys/fs/cgroup${CGROUP}/memory.stat | grep -E "^(anon|file|slab|sock) "
# anon + file + slab deve ≈ memory.current
```

---

> [!tip] Ordem recomendada
> Exercícios 1→2→3 são independentes e exploratórios. Faça-os em qualquer node disponível.
> Exercícios 4→5 se conectam — faça nessa ordem.
> Exercícios 6→7→8 são avançados — requerem entendimento sólido dos anteriores.
