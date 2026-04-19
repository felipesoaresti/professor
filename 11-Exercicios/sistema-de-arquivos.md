---
tags:
  - exercicios
  - linux
  - filesystem
  - fhs
  - inodes
tipo: exercicios
area: linux
conteudo: "[[01-Linux/sistema-de-arquivos]]"
trilha: "[[00-Trilha/linux]]"
---

# Exercícios — Sistema de Arquivos no Linux

> Conteúdo: [[01-Linux/sistema-de-arquivos]] | Exercícios: [[11-Exercicios/sistema-de-arquivos]] | Trilha: [[00-Trilha/linux]]

**Sem respostas aqui.** Execute, observe, retorne com o que encontrou.
Ambiente: qualquer node do homelab — `k8s-master (192.168.3.30)`, `worker1 (.31)`, `worker2 (.32)`.

---

## Exercício 1: Lendo o que o FHS realmente tem

**Contexto:** Você chegou em um servidor que não conhece. Precisa mapear rapidamente onde estão os dados relevantes antes de fazer qualquer mudança.

**Missão:** Explorar a estrutura do sistema e entender o que está montado, onde e com quais opções.

**Requisitos:**
- [ ] Listar todos os filesystems montados mostrando tipo e opções de montagem
- [ ] Identificar qual diretório está em qual dispositivo/filesystem
- [ ] Verificar se `/tmp` está montado como tmpfs (e o que isso significa na prática)
- [ ] Descobrir quais diretórios consomem mais espaço no sistema
- [ ] Verificar uso de inodes (não só de blocos) em todos os filesystems

**Verificação:**
```bash
findmnt --tree
df -hT
df -i
```

---

## Exercício 2: Inodes na prática

**Contexto:** Um desenvolvedor reporta que não consegue criar arquivos em `/var/cache/app/` mesmo com espaço em disco disponível. Você precisa diagnosticar.

**Missão:** Simular e diagnosticar o problema de inodes esgotados.

**Requisitos:**
- [ ] Criar um diretório de teste em `/tmp/inode-test/`
- [ ] Entender o número de inodes disponíveis naquele filesystem
- [ ] Criar um arquivo e inspecionar seu inode com `stat`
- [ ] Criar um hard link para esse arquivo e observar o que muda em `stat`
- [ ] Criar um symlink e observar a diferença
- [ ] Deletar o arquivo original e verificar o que acontece com o hard link
- [ ] Simular o que acontece quando se tenta criar um arquivo com o filesystem de inodes esgotados (documentar o erro)

**Verificação:**
```bash
df -i /tmp
stat /tmp/inode-test/<arquivo>
ls -li /tmp/inode-test/
```

---

## Exercício 3: find cirúrgico para SRE

**Contexto:** Um alerta disparou: o node `worker1` está com 85% de disco usado em `/var`. Você tem 10 minutos antes da janela de manutenção fechar.

**Missão:** Usar `find` para identificar o que está ocupando espaço e o que pode ser limpo com segurança.

**Requisitos:**
- [ ] Encontrar os 10 maiores arquivos em `/var` (sem considerar subdiretórios de sistemas de arquivos separados)
- [ ] Listar arquivos de log com mais de 7 dias de modificação em `/var/log`
- [ ] Encontrar arquivos temporários `.tmp` e `.swp` com mais de 24h em `/tmp`
- [ ] Identificar arquivos com permissão SUID em `/usr` (potencial risco de segurança)
- [ ] Calcular o espaço total que seria liberado se os logs antigos fossem removidos (sem remover ainda)

**Verificação:**
```bash
find /var -maxdepth 5 -type f -size +50M
find /var/log -type f -mtime +7
```

---

## Exercício 4: Arquivo deletado ainda segurando espaço

**Contexto:** Clássico de produção. `df` mostra o disco cheio, mas `du -sh /` mostra bem menos do que deveria. Você suspeita de arquivo deletado com fd aberto.

**Missão:** Reproduzir e resolver esse cenário no homelab.

**Requisitos:**
- [ ] Criar um arquivo grande de teste (use `fallocate -l 500M /tmp/bigfile.dat`)
- [ ] Abrir o arquivo em background mantendo o fd aberto (dica: `tail -f`)
- [ ] Deletar o arquivo com `rm`
- [ ] Confirmar que o espaço **não foi liberado** ainda (`df -h`, `du`)
- [ ] Identificar o processo que ainda tem o fd aberto (`lsof`)
- [ ] Recuperar o espaço sem matar o processo (truncar via `/proc`)
- [ ] Confirmar que o espaço foi liberado

**Verificação:**
```bash
lsof +L1
ls -la /proc/<PID>/fd/
```

---

## Exercício 5: Deploy com symlinks (zero downtime pattern)

**Contexto:** Você precisa implementar um padrão de deploy que permita rollback instantâneo sem parar a aplicação.

**Missão:** Implementar o padrão de releases com symlinks.

**Requisitos:**
- [ ] Criar a estrutura `/opt/myapp/releases/v1.0.0/` com arquivos de "aplicação" simulados
- [ ] Criar o symlink `/opt/myapp/current` apontando para `v1.0.0`
- [ ] Simular deploy de v1.1.0: criar a nova release, atualizar o symlink atomicamente
- [ ] Verificar que a troca foi atômica (dica: `strace` no comando de troca do symlink)
- [ ] Fazer rollback para v1.0.0
- [ ] Verificar com `readlink -f` que o caminho resolve corretamente em cada etapa

**Verificação:**
```bash
readlink -f /opt/myapp/current
strace -e rename,symlink,unlink ln -sfn /opt/myapp/releases/v1.1.0 /opt/myapp/current
ls -la /opt/myapp/
```

---

## Exercício 6: Permissões e segurança — o que chown/chmod realmente fazem

**Contexto:** Uma aplicação está falhando com "Permission denied" em `/var/lib/myapp/data/`. O processo roda como usuário `www-data`. Você precisa corrigir sem dar permissão demais.

**Missão:** Configurar permissões corretamente para uma aplicação web simulada.

**Requisitos:**
- [ ] Criar o usuário de sistema `appuser` (sem home, sem shell)
- [ ] Criar a estrutura `/var/lib/myapp/{data,logs,cache}/`
- [ ] Configurar: `data/` e `logs/` com dono `appuser`, leitura/escrita só para o dono
- [ ] Configurar: `cache/` com permissões para grupo também (o grupo de deploy precisa limpar)
- [ ] Aplicar permissões diferentes em arquivos vs diretórios dentro da árvore (sem usar `chmod -R` flat)
- [ ] Verificar com `stat` que as permissões em octal estão corretas
- [ ] Simular o acesso como o usuário `appuser` e confirmar que funciona

**Verificação:**
```bash
stat -c "%a %U:%G %n" /var/lib/myapp/data
sudo -u appuser touch /var/lib/myapp/data/test.txt
find /var/lib/myapp -exec stat -c "%a %U:%G %n" {} \;
```

---

## Exercício 7 (Avançado): inotify e eventos de filesystem

**Contexto:** Você precisa auditar mudanças em `/etc/` em um servidor de produção. Qualquer modificação em arquivos de configuração deve ser detectada imediatamente.

**Missão:** Configurar monitoramento de eventos de filesystem com inotifywait.

**Requisitos:**
- [ ] Instalar `inotify-tools` no node
- [ ] Configurar um watcher em `/etc/` que detecte modify, create, delete e moved
- [ ] Em outro terminal, fazer modificações em arquivos de `/etc/` e confirmar que são capturadas
- [ ] Verificar o limite atual de inotify watches do kernel
- [ ] Calcular: quantos watches um único pod Kubernetes consome em média (dica: olhar `/proc/sys/fs/inotify/max_user_watches` e comparar com o número de containers no node)
- [ ] Documentar o sintoma que aparece quando o limite é esgotado

**Verificação:**
```bash
inotifywait -m -r -e modify,create,delete,moved_from,moved_to /etc/ &
cat /proc/sys/fs/inotify/max_user_watches
cat /proc/sys/fs/inotify/max_user_instances
```

---

## Exercício 8 (Staff-level): Rastrear o que um processo faz no filesystem

**Contexto:** Uma aplicação está criando arquivos temporários em algum lugar e não limpando. O disco do node enche periodicamente. Você precisa encontrar onde.

**Missão:** Usar `strace` e `/proc` para mapear todas as interações com o filesystem de um processo real.

**Requisitos:**
- [ ] Escolher um processo em execução no node (sugestão: `kubelet`, `containerd`, ou qualquer daemon)
- [ ] Usar `strace -p <pid> -e trace=openat,creat,mkdir,unlink,rename` por 30 segundos
- [ ] Listar todos os fds abertos do processo via `/proc/<pid>/fd/`
- [ ] Identificar os cwd (current working directory) do processo via `/proc/<pid>/cwd`
- [ ] Comparar o que `lsof -p <pid>` mostra com o que você viu diretamente em `/proc`
- [ ] Documentar: qual a diferença de overhead entre usar `lsof` e ler `/proc` diretamente?

**Verificação:**
```bash
ls -la /proc/<pid>/fd/ | wc -l
ls -la /proc/<pid>/cwd
lsof -p <pid> | wc -l
strace -p <pid> -e trace=openat,creat -c   # summary de syscalls por 10s
```
