---
tags:
  - linux
  - filesystem
  - fhs
  - inodes
  - comandos
area: linux
tipo: conteudo
prerequisites:
  - "[[01-Linux/lvm-dispositivos-disco]]"
next:
  - "[[11-Exercicios/sistema-de-arquivos]]"
trilha: "[[00-Trilha/linux]]"
---

# Sistema de Arquivos no Linux

> Conteúdo: [[01-Linux/sistema-de-arquivos]] | Exercícios: [[11-Exercicios/sistema-de-arquivos]] | Trilha: [[00-Trilha/linux]]

---

## O que é e por que existe

No Linux, **tudo é arquivo**. Processos, dispositivos, sockets, pipes — todos expostos como arquivos. Essa abstração é o que torna o shell tão poderoso e o que permite redirecionar a saída de um processo direto para um dispositivo de rede.

O sistema de arquivos tem duas responsabilidades:

1. **Organização lógica:** estrutura hierárquica de diretórios (o que o usuário vê)
2. **Organização física:** como dados são gravados no disco (ext4, xfs, btrfs)

A camada que une os dois é o **VFS (Virtual Filesystem Switch)** — uma interface genérica do kernel que faz com que `cat /etc/passwd`, `cat /proc/cpuinfo` e `cat /dev/urandom` usem a mesma syscall `read()`, mesmo que por baixo sejam mecanismos completamente diferentes.

---

## Como funciona internamente

### O inode — identidade real de um arquivo

Todo arquivo tem um **inode**: uma estrutura no disco que armazena metadados. O nome do arquivo **não** está no inode — ele vive no diretório que aponta para aquele inode.

```
Diretório /etc:
  "passwd"  → inode 1048579
  "hosts"   → inode 1048612

Inode 1048579:
  tamanho: 2847 bytes
  permissões: 0644
  uid/gid: 0/0
  dono: root
  timestamps: atime, mtime, ctime
  ponteiros para blocos de dados no disco
```

Isso explica por que:
- **Hard links** são apenas múltiplos nomes apontando para o mesmo inode
- Renomear (`mv`) dentro do mesmo filesystem é instantâneo — só muda o nome no diretório
- Deletar um arquivo só libera espaço quando o link count chega a zero **e** nenhum processo tem o fd aberto

```bash
# ver o inode de um arquivo
stat /etc/passwd
ls -i /etc/passwd

# quantos hard links um arquivo tem
stat /bin/sh | grep Links
```

### VFS — a cola entre userspace e filesystem

```
Userspace:   open("/etc/passwd")
                   ↓
Kernel VFS:  busca dentry cache → resolve inode → chama ext4_open()
                   ↓
ext4:        lê bloco do disco → retorna file descriptor
```

O kernel mantém em memória:
- **dentry cache:** resolução de nomes → inodes (extremamente rápida em disco quente)
- **inode cache:** metadados de arquivos abertos
- **page cache:** conteúdo dos arquivos (dados em RAM)

```bash
# ver uso do page cache
grep -E "Cached|Buffers|SReclaimable" /proc/meminfo
```

---

## Estrutura de Diretórios — FHS

O **Filesystem Hierarchy Standard (FHS)** define onde cada tipo de dado vive. Não é burocracia: há razões operacionais concretas para cada divisão.

```
/
├── bin/      → binários essenciais do sistema (agora symlink para /usr/bin no Debian moderno)
├── sbin/     → binários de administração (agora symlink para /usr/sbin)
├── usr/      → programas instalados pelo sistema
│   ├── bin/  → binários de usuário
│   ├── lib/  → bibliotecas
│   └── local/→ programas instalados manualmente (fora do package manager)
├── etc/      → configurações do sistema (arquivos de texto, versionáveis)
├── var/      → dados variáveis (logs, caches, spool, banco de dados)
│   ├── log/  → logs do sistema e aplicações
│   ├── lib/  → estado persistente de aplicações (ex: /var/lib/docker)
│   └── tmp/  → temporários que sobrevivem a reboot
├── tmp/      → temporários apagados no reboot (montado como tmpfs)
├── home/     → diretórios dos usuários
├── root/     → home do root (separado por segurança)
├── proc/     → pseudoFS: estado do kernel e processos em tempo real
├── sys/      → pseudoFS: interface com dispositivos e kernel
├── dev/      → dispositivos (block devices, char devices, pipes)
├── run/      → dados de runtime (PIDs, sockets) — sempre em tmpfs
├── boot/     → kernel, initrd, bootloader
├── mnt/      → ponto de montagem temporário
├── media/    → dispositivos removíveis montados automaticamente
├── opt/      → software de terceiros instalado fora do FHS
└── srv/      → dados servidos por serviços (web, ftp)
```

### Por que a separação `/usr` importa

Historicamente, `/` ficava em um disco pequeno e rápido; `/usr` em um disco maior. Hoje, a separação ainda faz sentido operacionalmente:

- `/etc` → deve ser versionado no Git (Ansible, Puppet gerenciam aqui)
- `/var` → pode crescer sem controle (logs, docker layers) — montar em volume separado
- `/tmp` → tmpfs: RAM, rápido, limpo no reboot
- `/run` → tmpfs: nunca persistir dados críticos aqui

> [!warning]
> Em Kubernetes, `/var/lib/kubelet`, `/var/lib/docker` e `/var/log` são os maiores consumidores de disco nos nodes. Monitorar separadamente com alertas próprios.

### O que vive onde em produção

| Dado | Onde deve estar | Por quê |
|---|---|---|
| Binários da aplicação | `/usr/local/bin` ou `/opt/app` | Fora do package manager, não sobrescrito em update |
| Configuração | `/etc/app/` | Versionável, gerenciado por config management |
| Logs | `/var/log/app/` | Rotacionado pelo logrotate, monitorado pelo agent |
| Dados persistentes | `/var/lib/app/` ou volume separado | Sobrevive a updates, backup separado |
> [!info]
> `/proc` e `/sys` **não ocupam espaço em disco**. São gerados pelo kernel em tempo real. `cat /proc/cpuinfo` não lê nada do disco — o kernel constrói a resposta na hora.

---

## Na prática — comandos e exemplos reais

### Navegação e inspeção

```bash
# listar com detalhes (permissões, inode, tamanho, timestamps)
ls -lahit /etc/

# -i = inode number
# -h = human readable
# -a = inclui ocultos
# -t = ordenar por modificação

# onde estou e resolução de links
pwd
realpath /bin/sh         # resolve todos os symlinks

# árvore de diretórios (com profundidade controlada)
tree -L 2 /etc/
tree -L 3 -d /var/       # só diretórios
```

### Criação e remoção

```bash
# criar diretório (com pais se não existirem)
mkdir -p /opt/app/{config,logs,data}

# remover diretório vazio
rmdir /opt/app/logs

# remover com conteúdo (cuidado)
rm -rf /opt/app/old/

# criar arquivo vazio ou atualizar timestamp
touch /var/log/app/app.log

# criar arquivo com conteúdo
echo "Hello" > /tmp/test.txt
printf "linha1\nlinha2\n" > /tmp/multi.txt
```

> [!warning]
> `rm -rf` não tem lixeira. Em produção, prefira mover para um diretório temporário antes de deletar: `mv /opt/app/old /tmp/old-$(date +%s)` e deletar depois de confirmar.

### Cópia e movimento

```bash
# copiar arquivo
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak

# copiar preservando metadados (timestamps, permissões, owner)
cp -a /opt/app/ /opt/app-backup/

# copiar apenas se destino for mais antigo
cp -u /tmp/source.txt /tmp/dest.txt

# mover/renomear (dentro do mesmo FS: instantâneo — só muda entry no diretório)
mv /opt/app/old /opt/app/archive

# mover entre filesystems diferentes (copia + deleta)
mv /tmp/bigfile /var/data/bigfile
```

> [!tip]
> `mv` entre filesystems diferentes faz uma cópia física dos dados. Para arquivos grandes, use `rsync` com `--progress` para ter feedback e possibilidade de retomar.

### Links — hard vs soft

```bash
# hard link: segundo nome para o mesmo inode
ln /etc/passwd /tmp/passwd-hard
stat /etc/passwd    # Links: 2

# symlink: ponteiro para um caminho (pode apontar para inexistente)
ln -s /etc/nginx/nginx.conf /tmp/nginx-conf

# ver destino do symlink
readlink /tmp/nginx-conf
readlink -f /tmp/nginx-conf   # resolve recursivamente

# encontrar todos os hard links de um inode
find / -inum $(stat -c %i /etc/passwd) 2>/dev/null
```

**Hard link vs Symlink — quando usar cada:**

| | Hard Link | Symlink |
|---|---|---|
| Funciona entre filesystems | Não | Sim |
| Sobrevive se original for deletado | Sim | Não (dangling link) |
| Pode apontar para diretório | Não (exceto root) | Sim |
| Caso de uso | Backup eficiente, deduplicação | Versionamento de binários, `/usr/bin/python → python3` |

### Busca — find (e quando usar outras ferramentas)

```bash
# encontrar por nome
find /var/log -name "*.log"

# encontrar por tipo (f=file, d=directory, l=symlink)
find /tmp -type f

# encontrar por tamanho (arquivos > 100MB)
find /var -type f -size +100M

# encontrar por modificação recente (últimos 30 min)
find /etc -type f -mmin -30

# encontrar por permissão (arquivos SUID — risco de segurança)
find / -perm -4000 -type f 2>/dev/null

# encontrar e executar ação (remover arquivos .tmp com mais de 7 dias)
find /tmp -name "*.tmp" -mtime +7 -delete

# find + xargs (mais eficiente que -exec para muitos arquivos)
find /var/log -name "*.gz" -mtime +30 | xargs rm -f
```

> [!tip]
> `locate` é muito mais rápido que `find` para buscas por nome — usa índice do `updatedb`. Mas o índice pode estar desatualizado. Para arquivos recém-criados, `find` é necessário.

### Espaço em disco — df e du

```bash
# espaço por filesystem montado (-T mostra o tipo)
df -hT

# uso por diretório (ordenado por tamanho)
du -sh /var/* | sort -rh | head -20

# uso de inodes (quando df mostra espaço mas operações falham)
df -i

# identificar quem está consumindo espaço dentro de /var
du -h --max-depth=2 /var | sort -rh | head -20
```

> [!warning]
> `df` e `du` podem mostrar valores diferentes. Causa mais comum: arquivo deletado mas com **file descriptor ainda aberto** por algum processo. O espaço só é liberado quando o processo fecha o fd.
> ```bash
> # encontrar arquivos deletados ainda em uso
> lsof +L1
> ```

### Permissões

```bash
# formato octal
chmod 644 /etc/app/config.yaml    # rw-r--r--
chmod 755 /usr/local/bin/myscript # rwxr-xr-x
chmod 600 ~/.ssh/id_rsa           # rw------- (obrigatório para chaves SSH)

# recursivo (cuidado: aplicar em executáveis e diretórios da mesma vez é problemático)
chmod -R 644 /var/www/html/       # arquivos OK, diretórios precisam de x para entrar

# melhor: usar find para separar arquivos de diretórios
find /var/www/html -type f -exec chmod 644 {} \;
find /var/www/html -type d -exec chmod 755 {} \;

# mudar dono
chown app:app /var/lib/app/
chown -R www-data:www-data /var/www/html/

# ver permissões em octal
stat -c "%a %n" /etc/passwd
```

### stat — inspecionar metadados reais

```bash
stat /etc/passwd
# File: /etc/passwd
# Size: 2847       Blocks: 8     IO Block: 4096  regular file
# Device: fd00h    Inode: 1048579  Links: 1
# Access: (0644/-rw-r--r--)  Uid: 0  Gid: 0
# Access: 2026-04-15 10:00:00  ← atime (último acesso)
# Modify: 2026-03-01 09:00:00  ← mtime (último conteúdo alterado)
# Change: 2026-03-01 09:00:00  ← ctime (último metadado alterado — inclui chmod/chown)
```

> [!info]
> **atime, mtime, ctime** têm significados distintos:
> - `mtime` muda quando o conteúdo muda
> - `ctime` muda quando permissões ou owner mudam (mas **não** é "creation time")
> - Linux não armazena creation time por padrão (ext4 armazena como `crtime`, acessível via `debugfs`)

---

## Casos de uso e boas práticas

### Organizar aplicações fora do package manager

```bash
# padrão recomendado para apps customizadas
/opt/myapp/
├── bin/        # binários
├── conf/       # configurações (symlink para /etc/myapp/)
├── data/       # dados persistentes
└── logs/       # logs (symlink para /var/log/myapp/)

# adicionar ao PATH sem modificar /etc/environment globalmente
echo 'export PATH=$PATH:/opt/myapp/bin' > /etc/profile.d/myapp.sh
```

### Gerenciar versões com symlinks

```bash
# deploy pattern com symlinks (sem downtime)
/opt/app/
├── releases/
│   ├── v1.2.0/
│   └── v1.3.0/
└── current -> releases/v1.3.0/  # symlink

# atualizar sem downtime: apenas redireciona o symlink
ln -sfn /opt/app/releases/v1.4.0 /opt/app/current
# rollback instantâneo:
ln -sfn /opt/app/releases/v1.3.0 /opt/app/current
```

### Montar filesystems com opções de segurança

```bash
# /etc/fstab com boas práticas
/dev/sda1  /tmp      tmpfs  defaults,noexec,nosuid,nodev     0 0
/dev/sdb1  /var/data ext4   defaults,noatime,nofail          0 2

# noexec: impede execução de binários nessa partição
# nosuid: ignora bit SUID (reduz attack surface)
# noatime: elimina write de atime em cada leitura (performance)
# nofail: não travar o boot se o disco não montar
```

---

## Troubleshooting — cenários reais de produção

### "Disco cheio" mas df mostra espaço livre

```bash
# 1. verificar inodes esgotados
df -i /var

# 2. procurar arquivo deletado com fd aberto
lsof +L1

# 3. truncar o arquivo sem matar o processo (recupera espaço imediatamente)
lsof +L1 | grep deleted
truncate -s 0 /proc/<PID>/fd/<FD>
```

### Não consigo criar arquivo: "Permission denied" mesmo sendo root

```bash
# 1. verificar se o filesystem está montado como read-only
cat /proc/mounts | grep /var
dmesg | tail | grep remount

# 2. verificar atributos imutáveis (lsattr)
lsattr /var/log/
# se aparecer 'i': chattr -i /var/log/locked-file

# 3. verificar se é namespace de mount de container
ls -la /proc/$$/ns/mnt
```

### Encontrar o que está crescendo em disco

```bash
# monitorar em tempo real qual diretório está crescendo
watch -n5 'df -h && du -sh /var/* 2>/dev/null | sort -rh | head -10'

# o que foi modificado nos últimos 10 minutos
find /var -type f -mmin -10 | sort

# processo escrevendo mais no disco agora
iotop -o -b -n 3 | head -30
```

### Symlink que não funciona

```bash
# verificar se o destino existe
readlink -f /path/to/link   # se retornar vazio, destino não existe

# listar symlinks quebrados em um diretório
find /etc -type l ! -exec test -e {} \; -print

# verificar se é link relativo com problema de contexto
ls -la /path/to/link        # mostra o target como foi criado
# caminho relativo é resolvido a partir do diretório do link, não do pwd atual
```

---

## Nível avançado

### O que acontece em `rm` — por dentro

```bash
rm /tmp/arquivo.txt
```

1. Kernel decrementa o `link count` do inode
2. Se `link count == 0` **e** nenhum processo tem fd aberto: blocos marcados como livres no block bitmap
3. Se algum processo ainda tem o fd: inode persiste, `link count == 0`, mas dado ainda acessível via `/proc/<pid>/fd/<n>`
4. Só quando todos os fds fecham: kernel libera o inode e os blocos

```bash
# verificar: arquivo deletado ainda acessível via /proc
exec 3< /tmp/test.txt
rm /tmp/test.txt
cat /proc/$$/fd/3    # ainda funciona!
exec 3<&-            # fecha fd, agora o inode é liberado
```

### rename() — a syscall que faz mv ser atômico

`mv arquivo destino` dentro do mesmo filesystem chama a syscall `rename()`. Ela é **atômica**: ou o arquivo está no destino, ou no origem — nunca em estado intermediário. Isso é fundamental para deployments.

```bash
# confirmar com strace
strace -e rename mv /tmp/a /tmp/b
```

### /proc/mounts vs /etc/fstab vs findmnt

```bash
# /etc/fstab: configuração — o que *deve* ser montado no boot
# /proc/mounts: estado atual — o que *está* montado agora
# findmnt: a melhor forma de ver os dois lado a lado

findmnt --tree          # árvore completa
findmnt -t ext4,xfs     # filtrar por tipo
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS /var   # detalhes de um ponto
```

### Inotify — monitorar eventos de filesystem

```bash
# instalar inotify-tools
apt install inotify-tools

# monitorar eventos em /etc (útil para auditoria e debugging)
inotifywait -m -r -e modify,create,delete /etc/

# limite de watches do kernel (pode esgotar com muitos containers)
cat /proc/sys/fs/inotify/max_user_watches
sysctl fs.inotify.max_user_watches=524288  # aumentar
```

> [!warning]
> Em nodes Kubernetes com muitos pods, o limite de inotify watches pode esgotar. Sintoma: IDEs param de funcionar, ferramentas de hot-reload falham. Verificar com `sysctl fs.inotify.max_user_watches` e aumentar se necessário.

### rsync — copiar preservando tudo que cp não preserva

```bash
# rsync como substituto de cp -a, com progress e verificação
rsync -avh --progress /source/ /dest/

# excluir arquivos no destino que não existem mais na origem (sync real)
rsync -avh --delete /source/ /dest/

# dry run — ver o que seria feito sem fazer nada
rsync -avhn --delete /source/ /dest/

# copiar entre hosts (usa SSH por baixo)
rsync -avh -e "ssh -i ~/.ssh/id_ed25519" /local/path/ user@host:/remote/path/
```

---

## Referências

- `man hier` — documentação oficial da hierarquia de diretórios no Linux
- `man 7 inode` — estrutura de inodes
- `man find` — manual completo do find (filtros de tempo, permissão, tipo)
- `man 2 rename` — syscall de renomeação atômica
- FHS: [https://refspecs.linuxfoundation.org/fhs.shtml](https://refspecs.linuxfoundation.org/fhs.shtml)
- Brendan Gregg — "Systems Performance" 2ª ed., Cap. 8: File Systems
