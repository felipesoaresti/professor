---
tags:
  - exercicios
  - linux
  - rsync
  - transferencia-arquivos
  - backup
tipo: exercicios
area: linux
conteudo: "[[01-Linux/rsync]]"
trilha: "[[00-Trilha/linux]]"
---

# Exercícios — rsync

> Conteúdo: [[01-Linux/rsync]] | Trilha: [[00-Trilha/linux]]

---

## Exercício 1: Sync local com simulação prévia

**Contexto:** Você precisa sincronizar um diretório de configurações para um diretório de backup local. Antes de executar, quer ver exatamente o que será feito.

**Missão:** Criar uma estrutura de diretórios de teste e sincronizá-la com rsync.

**Requisitos:**
- [ ] Criar `/tmp/rsync-lab/origem/` com pelo menos 3 arquivos de conteúdos distintos
- [ ] Executar dry-run com `--itemize-changes` e interpretar cada coluna da saída
- [ ] Executar o sync real e confirmar que o destino tem os mesmos arquivos
- [ ] Modificar um arquivo na origem e rodar novamente — confirmar que só esse arquivo foi transferido (via `--stats`)

**Verificação:**
```bash
diff -r /tmp/rsync-lab/origem/ /tmp/rsync-lab/destino/
# Deve retornar vazio (sem diferenças)
```

---

## Exercício 2: A armadilha da barra

**Contexto:** Um colega sincronizou `/var/www/html` para `/backup/www` e o resultado foi `/backup/www/html/` em vez de `/backup/www/`. O site ficou quebrado após a restauração porque o caminho estava errado.

**Missão:** Demonstrar a diferença de comportamento com e sem barra no source.

**Requisitos:**
- [ ] Executar `rsync -av /tmp/rsync-lab/origem /tmp/rsync-lab/sem-barra/` e observar a estrutura criada
- [ ] Executar `rsync -av /tmp/rsync-lab/origem/ /tmp/rsync-lab/com-barra/` e observar a estrutura criada
- [ ] Documentar (comentário ou anotação) a diferença observada e quando usar cada forma
- [ ] Confirmar com `tree` ou `ls -la` em cada destino

**Verificação:**
```bash
ls /tmp/rsync-lab/sem-barra/
ls /tmp/rsync-lab/com-barra/
# A estrutura de subdiretórios deve ser diferente entre os dois
```

---

## Exercício 3: Sync remoto via SSH para worker1

**Contexto:** Você precisa distribuir arquivos de configuração para os workers do cluster k8s. Isso é feito por rsync sobre SSH, pois SCP não tem delta transfer.

**Missão:** Sincronizar um diretório do k8s-master para o k8s-worker1.

**Requisitos:**
- [ ] Criar `/tmp/rsync-config/` no k8s-master com 5 arquivos `.conf` de tamanhos variados
- [ ] Sincronizar para `felipe@192.168.3.31:/tmp/rsync-config-destino/` via rsync+SSH
- [ ] Usar `--stats` e identificar: quantos bytes foram transferidos vs tamanho total
- [ ] Modificar apenas um arquivo e sincronizar novamente — verificar que o speedup aumentou

**Verificação:**
```bash
ssh felipe@192.168.3.31 "ls -la /tmp/rsync-config-destino/"
# Deve listar os mesmos arquivos criados no master
```

---

## Exercício 4: rsync com --delete e filtros de exclusão

**Contexto:** Sistema de backup de `/etc` que exclui arquivos sensíveis e diretórios irrelevantes. O backup deve refletir exatamente o estado atual — arquivos removidos do source devem sumir do backup.

**Missão:** Criar um script de backup de `/etc` (ou de um diretório simulado) com exclusões e delete.

**Requisitos:**
- [ ] Sincronizar `/etc/` para `/tmp/backup-etc/` excluindo: `*.bak`, `*.swp`, `ssl/private/`, `shadow`, `gshadow`
- [ ] Confirmar com dry-run + `--itemize-changes` que os arquivos excluídos não aparecem na lista de transferência
- [ ] Criar um arquivo falso em `/tmp/backup-etc/` e verificar que `--delete` o remove na próxima sync
- [ ] Verificar o exit code do rsync após cada execução (`echo $?`) e documentar o significado

**Verificação:**
```bash
ls /tmp/backup-etc/ssl/private/ 2>/dev/null && echo "FALHOU — private/ foi copiado" || echo "OK — private/ excluído"
ls /tmp/backup-etc/ | grep -E "\.bak$|\.swp$" && echo "FALHOU — arquivos excluídos copiados" || echo "OK"
```

---

## Exercício 5: Snapshots incrementais com --link-dest

**Contexto:** O time de SRE quer 7 dias de retenção de backups sem multiplicar o espaço em disco. A técnica de rsync snapshots com hard links resolve isso.

**Missão:** Implementar um esquema de backup rotativo de 7 dias usando `--link-dest`.

**Requisitos:**
- [ ] Criar `/tmp/dados-source/` com ~10 arquivos de texto
- [ ] Executar o primeiro snapshot para `/tmp/snapshots/$(date +%Y-%m-%d)/`
- [ ] Modificar 2 arquivos na source e executar o segundo snapshot usando `--link-dest` no snapshot anterior
- [ ] Confirmar com `ls -lai` que arquivos não modificados compartilham o mesmo inode entre os dois snapshots
- [ ] Calcular o espaço real ocupado por cada snapshot com `du -sh` e explicar por que é menor que o total

**Verificação:**
```bash
# Comparar inodes de um arquivo não modificado entre os dois snapshots
ls -lai /tmp/snapshots/*/arquivo-nao-modificado.txt
# Os inodes devem ser iguais (hard link)
```

---

## Exercício 6: Medir impacto do -z em LAN vs WAN simulada

**Contexto:** Um desenvolvedor insiste em usar `-z` em todos os rsync "para ser mais rápido". Você suspeita que em LAN com dados já comprimidos, isso adiciona overhead de CPU sem benefício de banda.

**Missão:** Medir e comparar throughput com e sem `-z`.

**Requisitos:**
- [ ] Criar um arquivo de 100MB com dados compressíveis: `dd if=/dev/urandom bs=1M count=10 | gzip > /tmp/teste.gz && dd if=/tmp/teste.gz bs=1M count=10 >> /tmp/teste.gz` (ou use `dd if=/dev/zero`)
- [ ] Sincronizar para worker1 **sem** `-z` e registrar throughput (use `--stats` ou `time`)
- [ ] Sincronizar o mesmo arquivo **com** `-z` e registrar throughput
- [ ] Repetir com arquivo de texto puro (dados compressíveis) e comparar os resultados
- [ ] Documentar: em quais cenários `-z` faz sentido

**Verificação:**
```bash
# Comparar "bytes/sec" ou tempo total em cada execução
# A resposta correta depende dos números medidos — justifique com os dados
```

---

## Exercício 7: Recuperação de transferência interrompida

**Contexto:** Um arquivo de 500MB está sendo transferido para um worker via rsync. A conexão SSH cai no meio da transferência. Você precisa retomar sem retransmitir o que já foi enviado.

**Missão:** Simular uma transferência interrompida e retomá-la corretamente.

**Requisitos:**
- [ ] Criar um arquivo de 200MB: `dd if=/dev/urandom of=/tmp/bigfile.bin bs=1M count=200`
- [ ] Iniciar o rsync para worker1 **sem** `-P` — interromper com Ctrl+C no meio
- [ ] Verificar o estado do arquivo parcial no destino
- [ ] Reiniciar **com** `-P` e confirmar que o rsync retoma de onde parou (observar "% done" e bytes já transferidos)
- [ ] Comparar MD5 do arquivo de origem e destino após conclusão

**Verificação:**
```bash
md5sum /tmp/bigfile.bin
ssh felipe@192.168.3.31 "md5sum /tmp/bigfile.bin"
# Os hashes devem ser idênticos
```

---

## Exercício 8 (Staff-level): Diagnóstico de rsync lento em produção

**Contexto:** O backup diário do servidor de aplicação para o NFS do homelab está levando 3x mais que o esperado. Os arquivos têm ~15GB, mas estão mudando pouco (10-20% de delta por dia). O rsync está sendo executado com `-avz`.

**Missão:** Diagnosticar e otimizar o rsync, justificando cada mudança com dados.

**Requisitos:**
- [ ] Executar rsync com `--stats` e `--human-readable` para coletar baseline de transferência
- [ ] Identificar se `-z` está ajudando ou atrapalhando (tipo dos dados, utilização de CPU durante sync)
- [ ] Testar a cifra SSH mais rápida disponível e medir impacto: `ssh -Q cipher` para listar, depois `-e "ssh -c <cifra>"`
- [ ] Comparar mtime vs checksum mode: rodar com `-c` e medir quanto tempo extra leva
- [ ] Verificar se há muitos arquivos pequenos com `find /origem | wc -l` e se isso explica a lentidão
- [ ] Propor e testar pelo menos uma otimização com dados antes/depois
- [ ] Documentar: qual foi o gargalo real e como foi resolvido

**Verificação:**
```bash
# Comparar saída de --stats entre baseline e versão otimizada
# Métrica principal: "Total bytes sent" e tempo total (time rsync ...)
# Redução de > 20% no tempo total = exercício bem-sucedido
```

---

> [!tip] Antes de cada exercício que usa `--delete`, sempre faça um dry-run primeiro.
> `rsync -av --delete --dry-run --itemize-changes /origem/ /destino/`
> Uma linha com `*deleting` pode ser o arquivo errado sendo deletado.
