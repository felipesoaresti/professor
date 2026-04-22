---
tags:
  - labs
  - linux
  - ssh
  - openssh
tipo: lab
area: linux
conteudo: "[[01-Linux/ssh-openssh]]"
exercicios: "[[11-Exercicios/ssh-openssh]]"
trilha: "[[00-Trilha/linux]]"
---

# Lab — SSH / OpenSSH

---

## Lab 1 (Guiado): Setup completo de acesso seguro ao homelab

**Objetivo:** Sair de um acesso padrão com senha para uma configuração profissional com chaves ed25519, config, e ControlMaster.

**Tempo estimado:** 30-45 minutos

### Checkpoint 1 — Estado inicial

Documente o estado atual antes de qualquer mudança:

```bash
# na sua máquina local
ls -la ~/.ssh/
cat ~/.ssh/config 2>/dev/null || echo "sem config"

# no k8s-master
ssh felipe@192.168.3.30 "ls -la ~/.ssh/ && cat /etc/ssh/sshd_config | grep -E '^(PasswordAuth|PubkeyAuth|PermitRoot|Port)'"
```

Anote o que encontrar. Esta é sua baseline.

### Checkpoint 2 — Geração de chave

```bash
# gerar nova chave ed25519
ssh-keygen -t ed25519 -C "seu-usuario@sua-maquina-homelab" -f ~/.ssh/id_ed25519_homelab

# verificar o que foi gerado
ls -la ~/.ssh/id_ed25519_homelab*
```

**Pergunta:** qual a diferença entre o arquivo `.pub` e o arquivo sem extensão? O que acontece se você perder o arquivo sem extensão?

### Checkpoint 3 — Deploy da chave

```bash
# copiar a chave para todos os nodes
ssh-copy-id -i ~/.ssh/id_ed25519_homelab.pub felipe@192.168.3.30
ssh-copy-id -i ~/.ssh/id_ed25519_homelab.pub felipe@192.168.3.31
ssh-copy-id -i ~/.ssh/id_ed25519_homelab.pub felipe@192.168.3.32
```

Verificar que funcionou:
```bash
ssh -i ~/.ssh/id_ed25519_homelab felipe@192.168.3.30 "echo ok no master"
ssh -i ~/.ssh/id_ed25519_homelab felipe@192.168.3.31 "echo ok no worker1"
ssh -i ~/.ssh/id_ed25519_homelab felipe@192.168.3.32 "echo ok no worker2"
```

### Checkpoint 4 — SSH Config

Criar `~/.ssh/config` com o conteúdo abaixo. Adapte o usuário:

```sshconfig
Host k8s-master
    HostName 192.168.3.30
    User felipe
    IdentityFile ~/.ssh/id_ed25519_homelab

Host k8s-worker1
    HostName 192.168.3.31
    User felipe
    IdentityFile ~/.ssh/id_ed25519_homelab
    ProxyJump k8s-master

Host k8s-worker2
    HostName 192.168.3.32
    User felipe
    IdentityFile ~/.ssh/id_ed25519_homelab
    ProxyJump k8s-master

Host 192.168.3.*
    User felipe
    IdentityFile ~/.ssh/id_ed25519_homelab

Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ControlMaster auto
    ControlPath ~/.ssh/cm_%r@%h:%p
    ControlPersist 5m
    AddKeysToAgent yes
```

Testar:
```bash
time ssh k8s-master exit
time ssh k8s-master exit    # segunda deve ser < 100ms
ssh k8s-worker1 hostname
ssh k8s-worker2 hostname
```

### Checkpoint 5 — Hardening do servidor

No k8s-master:
```bash
ssh k8s-master
sudo vim /etc/ssh/sshd_config
```

Adicionar/ajustar:
```
PasswordAuthentication no
PermitRootLogin no
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 60
ClientAliveCountMax 3
```

Validar e recarregar:
```bash
sudo sshd -t && echo "config ok"
sudo systemctl reload sshd
```

**Não feche esta sessão ainda.** Abra **outra aba/terminal** e teste:
```bash
ssh k8s-master "echo ainda funciona"
```

Só feche a sessão anterior quando confirmar que o acesso continua funcionando.

---

## Lab 2 (Livre): Túnel SSH para acesso ao PostgreSQL

**Objetivo:** Acessar o PostgreSQL do homelab a partir da sua máquina local usando port forwarding SSH, sem expor a porta 5432 diretamente.

**Informações disponíveis:**
- PostgreSQL rodando no homelab (descubra onde exatamente)
- Seu ponto de entrada é o k8s-master
- Você tem acesso ao `psql` localmente

**Resultado esperado:** conseguir rodar `psql -h localhost -p 15432 -U postgres -c "SELECT current_database();"` a partir da sua máquina.

**Restrição:** não toque nos databases `controledb`, `promobot` ou `evolution`. Apenas conecte e rode queries de metadata.

O caminho é seu. Documente o que fez e qual foi o comando final.

---

## Lab 3 (Avançado): Simular e diagnosticar falha de autenticação

**Objetivo:** Reproduzir os erros mais comuns de SSH e treiná-los a diagnosticar de forma sistemática.

**Setup (aplicar no k8s-worker1):**

O instrutor (você mesmo) vai introduzir **uma das falhas abaixo** intencionalmente, depois tentar se diagnosticar sem olhar o que foi feito:

**Falha A — Permissões erradas:**
```bash
ssh k8s-worker1
chmod 755 ~/.ssh
```

**Falha B — authorized_keys corrompido:**
```bash
echo "lixo" >> ~/.ssh/authorized_keys
```

**Falha C — sshd em porta não-padrão:**
```bash
# sshd_config: Port 2222
sudo systemctl reload sshd
```

**Protocolo de diagnóstico que você deve seguir (sem olhar qual falha aplicou):**

1. `ssh -vvv user@host` — o que diz o verbose?
2. Verificar conectividade TCP: `nc -zv host 22`
3. Verificar status do sshd no servidor (via console out-of-band)
4. Verificar logs: `journalctl -u sshd --since "5 minutes ago"`
5. Verificar permissões em `~/.ssh/`
6. Verificar conteúdo de `authorized_keys`

**Entregável:** descrever qual falha foi, como identificou, e como corrigiu — passo a passo.

Depois rode com as três falhas e certifique-se de conseguir diagnosticar cada uma em menos de 2 minutos.
