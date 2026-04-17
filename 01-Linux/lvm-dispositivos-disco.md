---
tags:
  - linux
  - lvm
  - storage
  - block-devices
aliases:
  - LVM
  - Logical Volume Manager
area: linux
tipo: conteudo
next:
  - "[[11-Exercicios/lvm-dispositivos-disco]]"
trilha: "[[00-Trilha/linux]]"
---

# LVM e Dispositivos de Disco no Linux

> Exercícios: [[11-Exercicios/lvm-dispositivos-disco]] | Trilha: [[00-Trilha/linux]]

---

## O que é e por que existe

Sem LVM, o tamanho de uma partição é definido no momento da criação e mudar isso é arriscado e trabalhoso — exige reboot, ferramentas como `parted` ou `gparted`, e há risco de perda de dados.

**LVM (Logical Volume Manager)** é uma camada de abstração entre o kernel e os sistemas de arquivos que permite:
- Redimensionar volumes **sem reboot** e com o sistema rodando
- Criar **snapshots** consistentes de volumes em produção
- Agregar múltiplos discos físicos em um único volume lógico
- Thin provisioning — alocar espaço sob demanda

> [!info] LVM no seu homelab
> Os nodes K8s (k8s-master/worker1/worker2) rodam sobre VMs no Proxmox que usa LVM-thin (`vms-thin`, `local-lvm`). O CSI driver `nfs-homelab` usa NFS em cima do ext4 no pve1 — não LVM direto. Mas os próprios discos de VM do Proxmox são LVs.

---

## Arquitetura e Funcionamento Interno

LVM trabalha em três camadas:

```
┌─────────────────────────────────────────┐
│         Sistema de Arquivos             │  ← ext4, xfs, btrfs
│         /dev/mapper/vg0-lv_data         │
├─────────────────────────────────────────┤
│         Logical Volume (LV)             │  ← lv_data (tamanho flexível)
│         lv_root, lv_data, lv_swap       │
├─────────────────────────────────────────┤
│         Volume Group (VG)               │  ← vg0 (pool de espaço)
├────────────┬────────────┬───────────────┤
│   PV sda   │  PV sdb    │   PV sdc      │  ← Physical Volumes
│ /dev/sda2  │ /dev/sdb   │ /dev/sdc      │
└────────────┴────────────┴───────────────┘
```

### Physical Volume (PV)
- Um disco inteiro ou uma partição marcada para uso pelo LVM
- Dividido em **Physical Extents (PE)** — blocos de 4MB por padrão
- Contém metadata LVM nos primeiros setores (Label)

### Volume Group (VG)
- Pool de espaço formado pela soma dos PVs
- Unidade de alocação: **Physical Extent** (mesmo tamanho em todos os PVs do VG)
- Um VG pode crescer adicionando novos PVs

### Logical Volume (LV)
- "Partição virtual" criada dentro do VG
- Mapeada para PEs pelo **Device Mapper** do kernel
- Aparece em `/dev/mapper/` e `/dev/<vg>/<lv>`
- Tipos: linear, striped, mirrored, thin, snapshot, cache

### Device Mapper
O kernel usa o subsistema **dm (device-mapper)** para traduzir acesso ao LV em acesso aos PVs físicos. Cada LV vira um dispositivo em `/dev/mapper/`.

```bash
ls /dev/mapper/
# vg0-lv_root   vg0-lv_data   control
```

---

## Dispositivos de Bloco no Linux

Antes de criar LVM, é preciso entender a nomenclatura de dispositivos:

### Nomenclatura

| Dispositivo | Tipo |
|---|---|
| `/dev/sda`, `/dev/sdb` | SATA/SAS (SCSI) |
| `/dev/nvme0n1`, `/dev/nvme1n1` | NVMe |
| `/dev/vda`, `/dev/vdb` | Virtio (VMs KVM/QEMU) |
| `/dev/xvda` | Xen virtual disk |
| `/dev/sda1`, `/dev/sda2` | Partições do sda |
| `/dev/nvme0n1p1` | Partição 1 do nvme0n1 |

> [!tip] VMs no seu homelab
> Os VMs no Proxmox usam `virtio` — dentro da VM você verá `/dev/vda`. Os workers k8s-worker1/worker2 provavelmente têm `/dev/vda` como disco principal.

### Inspecionar discos

```bash
# Listar todos os block devices com hierarquia
lsblk
lsblk -f        # + filesystem type, UUID, mountpoint
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE,UUID

# Informações detalhadas de um disco
fdisk -l /dev/sda
parted /dev/sda print

# Ver tamanho real e modelo
cat /sys/block/sda/size          # tamanho em setores de 512 bytes
cat /sys/block/sda/queue/rotational   # 0=SSD, 1=HDD
```

### Tabelas de Partição

| | MBR | GPT |
|---|---|---|
| Limite de disco | 2TB | 9.4ZB |
| Partições | 4 primárias (+ extended) | 128 primárias |
| UEFI | Não | Sim |
| Backup da tabela | Não | Sim (no final do disco) |

> [!warning] MBR em produção
> Para discos > 2TB, use GPT obrigatoriamente. MBR não reconhece o espaço acima de 2TB.

---

## Na Prática — Comandos Essenciais

### Criar LVM do zero

```bash
# 1. Criar PV em um disco ou partição
pvcreate /dev/sdb
pvcreate /dev/sdc

# 2. Criar VG agregando os PVs
vgcreate vg_data /dev/sdb /dev/sdc

# 3. Criar LV dentro do VG
lvcreate -n lv_app -L 20G vg_data        # tamanho fixo
lvcreate -n lv_logs -l 100%FREE vg_data  # usar todo espaço restante
lvcreate -n lv_small -l 50%VG vg_data    # % do VG total

# 4. Criar filesystem no LV
mkfs.ext4 /dev/vg_data/lv_app
mkfs.xfs  /dev/vg_data/lv_logs

# 5. Montar
mkdir -p /data/app
mount /dev/vg_data/lv_app /data/app
```

### Inspecionar LVM

```bash
# PVs
pvs                      # resumo
pvdisplay /dev/sdb       # detalhado
pvscan                   # descobrir PVs no sistema

# VGs
vgs
vgdisplay vg_data
vgdisplay -v vg_data     # + LVs e PVs do VG

# LVs
lvs
lvdisplay /dev/vg_data/lv_app
lvs -o +devices          # mostrar PEs físicos mapeados
```

### Expandir LV (operação mais comum em produção)

```bash
# Cenário: lv_app está 90% cheio, precisa crescer 10GB

# 1. Verificar espaço disponível no VG
vgs vg_data              # VFree deve ser >= 10G

# 2. Expandir o LV
lvextend -L +10G /dev/vg_data/lv_app
# OU: expandir e fazer resize do FS em um comando
lvextend -L +10G -r /dev/vg_data/lv_app   # -r chama resize2fs/xfs_growfs

# 3. Se não usou -r, resize manual do filesystem
# Para ext4:
resize2fs /dev/vg_data/lv_app
# Para xfs (deve estar montado):
xfs_growfs /data/app
```

> [!warning] XFS não faz shrink
> XFS só cresce, nunca diminui. Para reduzir, precisa de backup/restore. ext4 suporta shrink mas requer `umount` primeiro.

### Adicionar disco ao VG (quando VFree está baixo)

```bash
# Disco novo chegou: /dev/sdd
pvcreate /dev/sdd
vgextend vg_data /dev/sdd
# Agora VFree aumentou e você pode fazer lvextend
```

### Remover PV do VG (movedata)

```bash
# Mover dados de /dev/sdb para outros PVs do VG (pode demorar)
pvmove /dev/sdb

# Remover PV do VG após pvmove
vgreduce vg_data /dev/sdb

# Remover label LVM do disco
pvremove /dev/sdb
```

---

## Snapshots LVM

Snapshot cria um LV que registra apenas os **blocos que mudaram** desde a criação — eficiente para backup consistente de volume em uso.

```bash
# Criar snapshot de lv_app com 5GB de COW space
lvcreate -n lv_app_snap -L 5G -s /dev/vg_data/lv_app

# Montar read-only para backup
mkdir /mnt/snap
mount -o ro /dev/vg_data/lv_app_snap /mnt/snap
rsync -av /mnt/snap/ /backup/

# Remover snapshot após backup
umount /mnt/snap
lvremove /dev/vg_data/lv_app_snap
```

> [!warning] Snapshot space
> Se o snapshot space (COW area) encher antes do backup terminar, o snapshot é invalidado. Monitore com `lvs -o +snap_percent`.

---

## Thin Provisioning

Thin LVM permite criar LVs maiores do que o espaço físico disponível — alocação acontece sob demanda.

```bash
# Criar thin pool de 100G
lvcreate -n thin_pool -L 100G --thinpool vg_data

# Criar thin LV de 500G (overcommit — apenas 100G existem fisicamente)
lvcreate -n lv_thin1 -V 500G --thin vg_data/thin_pool
```

> [!warning] Thin pool cheio = dados corrompidos
> Se o thin pool encher completamente, IOs falham e dados podem ser corrompidos. Monitore `lvs -o +data_percent,metadata_percent` e configure alertas antes de 80%.

---

## Casos de Uso e Boas Práticas

| Situação | Solução LVM |
|---|---|
| Disco cheio em produção sem reboot | `lvextend -r` para crescer ao vivo |
| Backup consistente de BD ativo | Snapshot LVM antes do backup |
| Migrar dados de disco físico falhando | `pvmove` para redistribuir |
| Provisionamento rápido de VMs | Thin pool (como o Proxmox faz) |
| RAID + flexibilidade | LVM sobre mdadm, ou LVM mirror |

**Boas práticas:**
- Não usar 100% do VG — manter ~10-15% para snapshots e movimentação
- Nomear VGs e LVs de forma descritiva: `vg_<host>`, `lv_<uso>`
- Monitorar `VFree` em VGs críticos (alerta em < 20%)
- Testar `pvmove` e `lvextend` em ambiente não-prod antes de produção

---

## Troubleshooting — Cenários Reais

### PV não encontrado ao bootar

```bash
# Erro: "Couldn't find device with uuid ..."
# Verificar PVs conhecidos vs presentes
pvs --all             # mostra PVs mesmo sem device presente
vgdisplay vg_data     # mostra se VG está degradado

# Se disco foi substituído com mesmo UUID (clone):
vgimportclone --basevgname vg_data_new /dev/sdb
```

### VG não ativa

```bash
vgchange -ay vg_data   # ativar VG e seus LVs
```

### Filesystem cheio mas LV tem espaço

```bash
df -h /data/app        # filesystem cheio
lvs vg_data/lv_app     # LV tem espaço?
# Se LV foi expandido mas FS não:
resize2fs /dev/vg_data/lv_app   # ext4
xfs_growfs /data/app            # xfs
```

### LV em uso (busy) na tentativa de remoção

```bash
fuser -vm /data/app    # quem está usando
lsof +D /data/app      # arquivos abertos
umount -l /data/app    # lazy umount como último recurso
```

---

## Nível Avançado

### LVM Cache (dm-cache)

Usar SSD como cache para LV em HDD:

```bash
# Criar cache LV no SSD
lvcreate -n lv_cache -L 50G vg_data /dev/ssd_pv

# Converter lv_app para usar cache
lvconvert --type cache --cachepool vg_data/lv_cache vg_data/lv_app
```

### LVMVDO (Virtual Data Optimizer)

Deduplicação e compressão inline em LVs — disponível em RHEL 8+ / kernel 5.3+:

```bash
lvcreate -n lv_vdo -L 50G --type vdo vg_data
```

### Leitura de metadata direta

```bash
# Ver metadata LVM raw (backup automático em /etc/lvm/backup/)
cat /etc/lvm/backup/vg_data

# Recuperar VG de metadata backup após desastre
vgcfgrestore -f /etc/lvm/backup/vg_data vg_data
```

### dm-multipath com LVM

Em ambientes com múltiplos caminhos ao storage (SAN):

```bash
# Ver mapeamento de multipath
multipath -ll

# LVM sobre multipath device
pvcreate /dev/mapper/mpatha
vgcreate vg_san /dev/mapper/mpatha
```

---

## Referências

- [LVM Admin Guide — Red Hat](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_and_managing_logical_volumes/)
- [Arch Linux Wiki — LVM](https://wiki.archlinux.org/title/LVM) — melhor referência prática
- [Linux Device Mapper](https://www.kernel.org/doc/html/latest/admin-guide/device-mapper/)
- [Brendan Gregg — Storage](https://www.brendangregg.com/linuxperf.html) — performance de I/O
- `man lvm`, `man lvmthin`, `man lvmsystemid`
