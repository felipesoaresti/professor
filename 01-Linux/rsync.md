---
tags:
  - linux
  - rsync
  - transferencia-arquivos
  - backup
area: linux
tipo: conteudo
prerequisites:
  - "[[01-Linux/sistema-de-arquivos]]"
  - "[[01-Linux/lvm-dispositivos-disco]]"
next:
  - "[[11-Exercicios/rsync]]"
trilha: "[[00-Trilha/linux]]"
---

# rsync

> Conteúdo: [[01-Linux/rsync]] | Exercícios: [[11-Exercicios/rsync]] | Trilha: [[00-Trilha/linux]]

## O que é e por que existe

`rsync` é uma ferramenta de sincronização de arquivos que transfere **apenas as diferenças** entre origem e destino — não o arquivo inteiro. Criado por Andrew Tridgell em 1996, nasceu da necessidade de sincronizar arquivos grandes via links lentos sem retransmitir dados que o destino já possui.

O problema que ele resolve: `cp` e `scp` sempre transferem o arquivo completo. Se você tem um backup de 10 GB e mudaram apenas 5 MB, o scp retransmite 10 GB. O rsync retransmite 5 MB (ou menos).

Usos em produção:
- Backup incremental de servidores
- Migração de dados entre hosts/storage
- Deploy de artefatos (antes do Docker predominar)
- Sincronização de NFS/filers entre datacenters
- Distribuição de conteúdo estático (CDN origin sync)

## Como funciona internamente — o Algoritmo rsync

O coração do rsync é o **rsync algorithm** (Tridgell & Mackerras, 1996). O problema a resolver: sincronizar arquivo A (origem) com arquivo B (destino) minimizando dados transferidos, sem acesso bidirecional simultâneo.

### Fase 1 — Receiver gera checksums dos blocos do destino

O lado destino divide o arquivo em blocos de tamanho fixo (default: ~700 bytes em versões antigas, atualmente calculado dinamicamente com base no tamanho do arquivo). Para cada bloco, calcula:
- **Rolling checksum (Adler-32 modificado):** checksum fraco, computacionalmente barato, permite sliding window
- **MD4/MD5 checksum:** checksum forte, computacionalmente caro, usado apenas para confirmar match

O receptor envia essa lista de checksums para o transmissor.

### Fase 2 — Sender encontra blocos em comum

O transmissor faz um **sliding window** no arquivo de origem, um byte por vez, calculando o rolling checksum de cada janela. Se o rolling checksum bate com algum bloco da lista, confirma com o MD4/MD5. Se os dois batem: bloco idêntico, não precisa transferir.

O resultado é uma lista de instruções:
- "copia o bloco N do destino" (já existe lá)
- "insere estes bytes literais" (novo ou modificado)

### Fase 3 — Receiver reconstrói o arquivo

Usando as instruções, o receptor monta o novo arquivo: copia blocos locais de B que continuam iguais + insere os bytes literais enviados pelo transmissor.

```
Origem:  [AAAA][BBBB][CCCC][DDDD]
Destino: [AAAA][XXXX][CCCC]

Checksums do destino → transmissor
Sliding window na origem:
  AAAA → match bloco 0 do destino → não envia
  BBBB → sem match → envia literal
  CCCC → match bloco 2 do destino → não envia
  DDDD → sem match → envia literal

Transmitido: literal(BBBB) + literal(DDDD)
Destino reconstrói: bloco0_local + BBBB + bloco2_local + DDDD
```

> [!info] Por que rolling checksum?
> O Adler-32 modificado permite calcular o checksum de uma janela deslizante em O(1): dado o checksum de bytes [i..i+n], o checksum de [i+1..i+n+1] é derivável aritmeticamente sem reprocessar todos os bytes. Isso torna o sliding window eficiente.

### Detecção de mudanças — o que o rsync compara por padrão

Por padrão, rsync **não calcula checksum do arquivo inteiro**. Ele compara:
1. **Tamanho do arquivo** — se diferente, arquivo mudou
2. **mtime (modification time)** — se diferente, arquivo mudou

Só se as duas condições forem iguais é que assume que o arquivo é idêntico. Isso é rápido mas pode falhar se mtime foi preservado artificialmente.

Com `--checksum` (`-c`), calcula MD5 do arquivo inteiro antes de decidir sincronizar — mais confiável, muito mais lento.

## Na prática — comandos e exemplos reais

### Sintaxe básica

```bash
rsync [opções] origem destino
```

A barra no final importa:
```bash
rsync -av /dados/logs/ /backup/logs/   # copia o CONTEÚDO de logs/ para dentro de backup/logs/
rsync -av /dados/logs  /backup/logs/   # copia a PASTA logs como subdiretório: backup/logs/logs/
```

> [!warning] A barra é uma das fontes mais comuns de erro com rsync.
> `rsync src/ dst/` → conteúdo de src vai para dst
> `rsync src dst/` → src (a pasta inteira) vai para dentro de dst

### Flags essenciais

| Flag | Significado |
|---|---|
| `-a` / `--archive` | Modo arquivo: `-rlptgoD` — recursivo, links, permissões, timestamps, owner, group, devices |
| `-v` / `--verbose` | Verbose — lista arquivos sendo processados |
| `-z` / `--compress` | Compressão em trânsito (útil em links lentos, inútil em LAN com dados já comprimidos) |
| `-P` | `--partial --progress` — mostra progresso e mantém arquivos parcialmente transferidos |
| `-n` / `--dry-run` | Simulação — mostra o que seria feito sem executar |
| `-e` | Especifica shell remoto: `-e ssh` ou `-e "ssh -p 2222"` |
| `--delete` | Remove no destino arquivos que não existem mais na origem |
| `--exclude` | Exclui padrão: `--exclude='*.log'` |
| `--include` | Inclui padrão (combinado com exclude) |
| `--bwlimit` | Limita banda em KB/s: `--bwlimit=10000` |
| `--checksum` | Compara por MD5 ao invés de mtime+size |
| `--inplace` | Atualiza arquivo in-place (sem arquivo temporário) — relevante para arquivos grandes |
| `--sparse` | Suporte a arquivos sparse (ex: disk images) |
| `--hard-links` | Preserva hard links (`-H`) |
| `--acls` / `-A` | Preserva ACLs POSIX |
| `--xattrs` / `-X` | Preserva extended attributes |

### Exemplos práticos

**Backup local com delete:**
```bash
rsync -av --delete /var/www/html/ /backup/www/
```

**Sync remoto via SSH:**
```bash
rsync -avz -e ssh /dados/app/ usuario@192.168.3.31:/backup/app/
```

**Sync remoto com porta SSH customizada:**
```bash
rsync -avz -e "ssh -p 2222 -i ~/.ssh/id_ed25519" /dados/ backup@host:/dados/
```

**Dry-run antes de executar com --delete:**
```bash
rsync -av --delete --dry-run /origem/ /destino/
```

**Excluir padrões:**
```bash
rsync -av \
  --exclude='*.log' \
  --exclude='*.tmp' \
  --exclude='.git/' \
  --exclude='node_modules/' \
  /app/src/ /backup/src/
```

**Include + exclude (ordem importa — primeiro match ganha):**
```bash
rsync -av \
  --include='*.conf' \
  --exclude='*' \
  /etc/ /backup/etc/
# Só copia arquivos .conf, exclui todo o resto
```

**Limitar banda (útil em produção para não saturar link):**
```bash
rsync -avz --bwlimit=50000 /dados/ remoto:/dados/  # limita a ~50 MB/s
```

**Mostrar progresso em transferências grandes:**
```bash
rsync -av --progress /dados/db-dump.sql.gz remoto:/backup/
# ou com -P (partial + progress):
rsync -avP /dados/db-dump.sql.gz remoto:/backup/
```

**Verificar diferenças sem transferir (itemize):**
```bash
rsync -avn --itemize-changes /origem/ /destino/
```

Saída do `--itemize-changes`:
```
>f.st...... arquivo.txt   # enviado, tamanho e timestamp diferentes
.f..t...... outro.txt     # timestamp atualizado, conteúdo igual
*deleting   removido.txt  # seria deletado (com --delete)
```

Formato: `YXcstpoguax` — cada posição indica um atributo (Y=tipo de transferência, X=tipo de arquivo, c=checksum, s=size, t=timestamp, p=permissions, o=owner, g=group, u=ACL, a=xattr).

### rsync como daemon

O rsync pode rodar como daemon (porta 873), sem precisar de SSH:

```bash
# /etc/rsyncd.conf
[backup]
    path = /backup
    read only = no
    list = yes
    uid = backup
    gid = backup
    hosts allow = 192.168.3.0/24
    auth users = rsyncuser
    secrets file = /etc/rsyncd.secrets
```

```bash
rsync rsyncuser@192.168.3.30::backup /local/destino/
```

> [!warning] rsync daemon sem SSH não cifra os dados em trânsito — autenticação é por senha mas payload é plaintext. Em redes não confiáveis, sempre use SSH.

### rsync sobre SSH com chave (sem senha — automação)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/rsync_key -N ""
ssh-copy-id -i ~/.ssh/rsync_key.pub usuario@192.168.3.31

# No cron ou script:
rsync -az -e "ssh -i ~/.ssh/rsync_key" /dados/ usuario@192.168.3.31:/backup/
```

Para restringir o que a chave pode fazer no authorized_keys:
```
command="rsync --server --sender -logDtprze.iLsfx . /backup/",no-port-forwarding,no-X11-forwarding,no-agent-forwarding ssh-ed25519 AAAA...
```

## Casos de uso e boas práticas

### Backup incremental com snapshot

rsync não faz snapshots nativamente, mas combinado com hard links simula backup incremental eficiente (técnica de "rsync snapshots"):

```bash
#!/bin/bash
DEST=/backup
SRC=/dados
DATE=$(date +%Y-%m-%d)

rsync -av --delete \
  --link-dest=$DEST/ultimo \
  $SRC/ $DEST/$DATE/

# Atualiza ponteiro "ultimo"
rm -f $DEST/ultimo
ln -s $DEST/$DATE $DEST/ultimo
```

`--link-dest` faz hard links para arquivos iguais ao snapshot anterior → cada snapshot parece completo mas ocupa só o espaço das diferenças.

### Migração de servidor

```bash
# Na nova máquina, puxando do servidor antigo:
rsync -avz --exclude='/proc' --exclude='/sys' \
  --exclude='/dev' --exclude='/run' \
  --exclude='/tmp' --exclude='/mnt' \
  root@192.168.3.30:/ /

# Segundo pass para pegar mudanças durante o primeiro:
rsync -avz --delete --exclude='/proc' ... root@192.168.3.30:/ /
```

### Logs de transferência

```bash
rsync -av --log-file=/var/log/rsync.log /dados/ /backup/
```

### Verificação pós-transferência

```bash
# Compara origem e destino por checksum (lento mas definitivo)
rsync -avn --checksum /origem/ /destino/
# Se não aparecer nada para transferir → conteúdo idêntico
```

## Troubleshooting — cenários reais de produção

### Transferência muito lenta

```bash
# Diagnóstico: medir throughput real
rsync -avP --stats /dados/ remoto:/backup/
```

Saída de `--stats`:
```
Number of files: 1,234
Number of regular files transferred: 89
Total file size: 10.23G bytes
Total transferred file size: 234.56M bytes
Literal data: 198.34M bytes
Matched data: 36.22M bytes     ← dados aproveitados do destino (algoritmo delta)
File list size: 45.67K
Total bytes sent: 201.23M
Total bytes received: 12.45K
speedup is 50.74               ← razão entre tamanho total e tamanho transferido
```

Se `speedup` for baixo (próximo de 1): o destino não tem versões anteriores para reaproveitar, ou os arquivos mudam muito.

Causas de lentidão:
- `-z` em LAN com dados já comprimidos → adiciona overhead de CPU sem ganho
- Muitos arquivos pequenos → overhead de handshake por arquivo supera transferência
- SSH com cifra pesada → testar `-e "ssh -c aes128-gcm@openssh.com"` ou `-e "ssh -c chacha20-poly1305@openssh.com"`

Para muitos arquivos pequenos considerar:
```bash
# Tar + rsync pipe (uma única stream SSH)
tar -czf - /dados/ | ssh remoto "tar -xzf - -C /backup/"
```

### Permissões incorretas no destino

```bash
# -a preserva permissões — pode dar problema se owner não existe no destino
rsync -rltz --no-owner --no-group /dados/ /backup/
# ou especificar owner no destino:
rsync -av --chown=www-data:www-data /dados/ /backup/
```

### Arquivo sendo modificado durante sync

```bash
# rsync pode pegar estado inconsistente de arquivo em escrita
# Para databases ou arquivos grandes em uso:
rsync -av --inplace --partial /dados/bigfile /backup/
# --inplace: atualiza o arquivo diretamente (não cria temporário)
# evita que o destino fique com arquivo temporário se interrompido
```

Para consistency real: snapshot LVM/ZFS antes do rsync.

### rsync falha com "vanished file" (erro 24)

```bash
rsync: [sender] link_stat "..." failed: No such file or directory (2)
rsync warning: some files vanished before they could be transferred (code 24)
```

Arquivo existia na listagem mas sumiu antes de ser transferido (log rotacionado, processo deletou). Código de saída 24 é warning, não erro fatal. Para ignorar em scripts:

```bash
rsync -av /dados/ /backup/ || [ $? -eq 24 ]
```

### Debugging de permissão negada

```bash
rsync -avvv /dados/ remoto:/backup/  # triple verbose — mostra handshake SSH e negociação
```

### Verificar o que o rsync faria antes de rodar

```bash
# Sempre dry-run antes de --delete em produção
rsync -av --delete --dry-run --itemize-changes /origem/ /destino/ | head -50
```

## Nível avançado — flags menos conhecidas e edge cases

### `--checksum-choice` — algoritmo de checksum

```bash
rsync --checksum-choice=xxh128 -av /dados/ /backup/  # xxHash — muito mais rápido que MD5
```

rsync 3.2+ suporta: `md4`, `md5`, `sha1`, `xxh64`, `xxh128`, `xxhash`.

### Transferência paralela com GNU parallel

rsync é single-threaded por design. Para paralelizar:

```bash
ls /dados/ | parallel -j4 rsync -av /dados/{}/ remoto:/backup/{}/
```

### `--files-from` — sync seletivo de lista

```bash
find /dados -name "*.conf" -mtime -7 > /tmp/changed.txt
rsync -av --files-from=/tmp/changed.txt / remoto:/backup/
```

### `--backup` e `--backup-dir` — histórico de versões

```bash
rsync -av --backup --backup-dir=/backup/old-$(date +%Y%m%d) \
  --delete /dados/ /backup/atual/
# Arquivos sobrescritos ou deletados vão para /backup/old-YYYYMMDD/
```

### Preservar sparse files (disk images, database files)

```bash
rsync -avS /vms/*.qcow2 /backup/vms/  # -S = --sparse
# Sem -S, arquivos sparse são expandidos e ocupam espaço real no destino
```

### rsync com ACLs e xattrs (relevante para SELinux/AppArmor)

```bash
rsync -avAX /dados/ /backup/  # -A = ACLs, -X = xattrs
# Necessário para migrar sistemas com SELinux contexts ou Capabilities (setcap)
```

### `--max-size` e `--min-size` — filtrar por tamanho

```bash
rsync -av --max-size=100M /dados/ /backup/  # ignora arquivos > 100MB
rsync -av --min-size=1k /dados/ /backup/    # ignora arquivos < 1KB (evita arquivos de controle)
```

### Detecção de loop (rsync dentro de rsync no destino)

Se origem e destino se sobrepõem:
```bash
rsync -av --exclude='/backup/' / /backup/  # essencial excluir o destino da origem
```

### Exit codes relevantes

| Código | Significado |
|---|---|
| 0 | Sucesso |
| 1 | Erro de sintaxe |
| 11 | Erro de I/O |
| 12 | Erro no protocolo de dados |
| 23 | Alguns arquivos não foram transferidos (permissão) |
| 24 | Alguns arquivos sumiram antes de ser transferidos (warning) |
| 25 | Transferência interrompida pela opção `--max-delete` |
| 30 | Timeout |

### Resumir transferência interrompida

```bash
rsync -avP /dados/bigfile remoto:/backup/bigfile
# -P = --partial --progress
# Se interrompido e rodado novamente, continua de onde parou
```

> [!tip] Em transferências muito grandes (>10GB), sempre use `-P`. Sem ele, arquivos parcialmente transferidos são deletados e o rsync recomeça do zero.

### rsync vs cp vs scp — quando usar cada um

| Situação | Ferramenta |
|---|---|
| Cópia local, arquivo único | `cp` |
| Transferência remota, arquivo único, sem versão anterior no destino | `scp` |
| Sync incremental, backup, qualquer coisa que possa já existir no destino | `rsync` |
| Muitos arquivos pequenos, transferência remota | `rsync` com `-z` em WAN, sem `-z` em LAN |
| Consistência transacional obrigatória | snapshot (LVM/ZFS) + rsync |

## Referências

- `man rsync` — documentação completa (extensa, leia a seção OPTIONS)
- [rsync algorithm](https://rsync.samba.org/tech_report/) — paper original de Tridgell & Mackerras
- [rsync.samba.org](https://rsync.samba.org/) — site oficial
- Brendan Gregg — [Linux Performance Tools](https://www.brendangregg.com/linuxperf.html) (contexto de I/O e transferência)
- [rsync man page online](https://download.samba.org/pub/rsync/rsync.1)
