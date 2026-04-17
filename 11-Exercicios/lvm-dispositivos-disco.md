---
tags:
  - exercicios
  - linux
  - lvm
  - storage
tipo: exercicios
area: linux
conteudo: "[[01-Linux/lvm-dispositivos-disco]]"
trilha: "[[00-Trilha/linux]]"
---

# Exercícios — LVM e Dispositivos de Disco

> Conteúdo: [[01-Linux/lvm-dispositivos-disco]] | Trilha: [[00-Trilha/linux]]
> Infraestrutura: k8s-master (192.168.3.30), k8s-worker1 (192.168.3.31), k8s-worker2 (192.168.3.32)
> Referência: `man lvm`, `man lvcreate`, Arch Wiki LVM

> [!warning] Atenção
> Todos os exercícios devem ser feitos em disco **extra/não-usado**. Nunca no disco do sistema (`/dev/vda` principal). Use um disco adicional na VM ou um arquivo de loop device.

---

## Exercício 1: Reconhecimento de Dispositivos de Bloco

**Contexto:** Você acabou de assumir a operação de um servidor Linux e precisa mapear o storage disponível antes de qualquer intervenção.

**Missão:** Em qualquer node do homelab (ex: `k8s-master`), levantar um inventário completo dos dispositivos de bloco e seu estado atual.

**Requisitos:**
- [ ] Listar todos os block devices com tipo, tamanho e filesystem
- [ ] Identificar quais partições estão montadas e onde
- [ ] Verificar se há algum PV, VG ou LV LVM existente no sistema
- [ ] Identificar o UUID de pelo menos um filesystem montado
- [ ] Confirmar o tipo de tabela de partição (MBR ou GPT) do disco principal

**Verificação:**
```bash
lsblk -f
```

---

## Exercício 2: Loop Device como Disco Fictício

**Contexto:** Você precisa testar configurações de LVM mas não tem disco extra disponível. Loop devices são a solução — criam discos virtuais a partir de arquivos.

**Missão:** Criar dois "discos" virtuais usando loop devices e prepará-los para uso com LVM.

**Requisitos:**
- [ ] Criar dois arquivos de 500MB cada em `/tmp/`
- [ ] Associar cada arquivo a um loop device (`/dev/loop0` e `/dev/loop1` ou próximos disponíveis)
- [ ] Confirmar que os loop devices aparecem como block devices no `lsblk`
- [ ] Criar um PV em cada loop device
- [ ] Confirmar os PVs criados com `pvs`

**Verificação:**
```bash
losetup -l
pvs
```

> [!tip]
> `losetup --find --show /tmp/arquivo.img` cria e associa em um comando.

---

## Exercício 3: Construir um VG e LV do Zero

**Contexto:** A equipe de infra precisa de um volume dedicado para logs de aplicação. Você tem dois loop devices disponíveis e quer criar um VG consolidado com um LV de 600MB.

**Missão:** Criar um VG chamado `vg_lab` usando os dois loop devices, e dentro dele um LV chamado `lv_logs` com 600MB, formatado em ext4 e montado em `/mnt/lv_logs`.

**Requisitos:**
- [ ] VG `vg_lab` criado com os dois PVs
- [ ] LV `lv_logs` com 600MB criado dentro do `vg_lab`
- [ ] Filesystem ext4 criado no LV
- [ ] LV montado em `/mnt/lv_logs`
- [ ] Escrever um arquivo de teste em `/mnt/lv_logs/teste.txt`

**Verificação:**
```bash
vgdisplay vg_lab
lvdisplay /dev/vg_lab/lv_logs
df -h /mnt/lv_logs
```

---

## Exercício 4: Expandir LV ao Vivo

**Contexto:** O LV `lv_logs` está 80% cheio (você vai simular isso). A equipe pede para crescer **200MB** sem desmontar o volume.

**Missão:** Expandir o LV `lv_logs` em 200MB com o filesystem montado e em uso.

**Requisitos:**
- [ ] Simular uso: criar um arquivo de 400MB em `/mnt/lv_logs/` com `dd`
- [ ] Confirmar que o LV está "cheio" antes do resize
- [ ] Expandir o LV em 200MB **sem desmontar**
- [ ] Confirmar que o filesystem cresceu (df deve mostrar novo tamanho)
- [ ] O arquivo de teste criado anteriormente ainda deve estar intacto

**Verificação:**
```bash
df -h /mnt/lv_logs
ls -lh /mnt/lv_logs/
```

> [!tip]
> `lvextend -r` faz o resize do filesystem automaticamente junto com o LV.

---

## Exercício 5: Snapshot e Restauração

**Contexto:** Antes de uma operação crítica, você precisa criar um snapshot do `lv_logs` para poder reverter se algo der errado.

**Missão:** Criar um snapshot do `lv_logs`, simular uma "corrupção" (deletar um arquivo), e restaurar o volume ao estado anterior via snapshot.

**Requisitos:**
- [ ] Criar snapshot `lv_logs_snap` com 200MB de COW space
- [ ] Confirmar que o snapshot existe com `lvs`
- [ ] **Simular corrupção:** deletar o arquivo `teste.txt` do volume original
- [ ] Montar o snapshot em `/mnt/snap` em modo read-only e confirmar que `teste.txt` ainda está lá
- [ ] Desmontar e remover o snapshot após validação

**Verificação:**
```bash
lvs -o +snap_percent vg_lab
ls /mnt/snap/
```

---

## Exercício 6: Adicionar PV ao VG (VFree crítico)

**Contexto:** O VG `vg_lab` está sem espaço livre (VFree = 0). Um novo disco chegou e precisa ser integrado ao VG sem parar o serviço.

**Missão:** Criar um terceiro loop device de 500MB, adicioná-lo ao VG `vg_lab` como novo PV, e usar o espaço para crescer o `lv_logs` mais 300MB.

**Requisitos:**
- [ ] Criar terceiro loop device `/tmp/disco3.img`
- [ ] Criar PV no novo loop device
- [ ] Adicionar o PV ao VG `vg_lab` com `vgextend`
- [ ] Confirmar que VFree aumentou
- [ ] Expandir `lv_logs` usando o novo espaço

**Verificação:**
```bash
vgs vg_lab
pvs
```

---

## Exercício 7 (Avançado): Remover PV com pvmove

**Contexto:** Um dos loop devices está com setores com defeito e precisa ser substituído. Você precisa evacuar os dados dele para os outros PVs sem perder nada.

**Missão:** Remover `/dev/loop0` (ou o primeiro PV) do VG usando `pvmove`, e depois retirar o PV do VG.

**Requisitos:**
- [ ] Confirmar distribuição de dados atual nos PVs com `pvs -o +pvseg_all`
- [ ] Executar `pvmove` no PV a ser removido
- [ ] Confirmar que o PV ficou com 0 PEs alocados
- [ ] Remover o PV do VG com `vgreduce`
- [ ] Remover a label LVM do device com `pvremove`
- [ ] VG `vg_lab` ainda deve funcionar com os LVs intactos

**Verificação:**
```bash
pvs
vgdisplay vg_lab
cat /mnt/lv_logs/teste.txt
```

---

## Exercício 8 (CKA-level): Troubleshooting — VG Degradado

**Contexto:** Você recebe o alerta: *"VG vg_data not found"* após um reboot de emergência. Um disco físico foi desconectado acidentalmente.

**Missão:** Simular o cenário de PV ausente e recuperar o VG usando metadata backup.

**Requisitos:**
- [ ] Desassociar um dos loop devices com `losetup -d` (simular disco removido)
- [ ] Tentar ativar o VG e documentar o erro exato
- [ ] Localizar o backup de metadata em `/etc/lvm/backup/`
- [ ] Reconectar o loop device (simular disco reinserido)
- [ ] Reativar o VG com `vgchange -ay`
- [ ] Descrever o que aconteceria se o disco fosse irrecuperável (resposta teórica)

**Verificação:**
```bash
vgchange -ay vg_lab
lvs
```

---

## Limpeza (após exercícios)

```bash
umount /mnt/lv_logs
umount /mnt/snap 2>/dev/null
lvremove -f /dev/vg_lab/lv_logs
vgremove -f vg_lab
pvremove /dev/loop0 /dev/loop1 /dev/loop2 2>/dev/null
losetup -D   # desassociar todos os loop devices
rm -f /tmp/disco1.img /tmp/disco2.img /tmp/disco3.img
```
