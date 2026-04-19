---
tags:
  - networking
  - linux
  - ssh
  - openssh
  - segurança
area: networking
tipo: conteudo
prerequisites:
  - "[[01-Linux/lvm-dispositivos-disco]]"
next:
  - "[[11-Exercicios/ssh-openssh]]"
  - "[[12-Labs/ssh-openssh]]"
trilha: "[[00-Trilha/networking]]"
---

# SSH / OpenSSH

## O que é e por que existe

SSH (Secure Shell) é um protocolo de rede criptografado que substitui o trio inseguro `telnet` / `rsh` / `rcp`. Criado em 1995 por Tatu Ylönen após um ataque de sniffing na sua rede universitária, hoje é o principal mecanismo de acesso remoto e automação em ambientes Unix/Linux.

**OpenSSH** é a implementação open-source dominante — mantida pelo projeto OpenBSD, portada para Linux e a maioria dos sistemas. Compõe-se de:

| Binário | Função |
|---|---|
| `sshd` | daemon — escuta conexões, autentica, faz fork por sessão |
| `ssh` | cliente — inicia conexões |
| `scp` | cópia de arquivos via SSH (usa protocolo SCP sobre SSH) |
| `sftp` | subsistema de transferência de arquivos (protocolo SFTP sobre SSH) |
| `ssh-keygen` | geração e gerenciamento de chaves |
| `ssh-agent` | agente de chaves em memória |
| `ssh-add` | adiciona chaves ao agent |
| `ssh-copy-id` | copia chave pública para o servidor |

---

## Como funciona internamente

### Camadas do protocolo SSH-2

```
┌──────────────────────────────────────────┐
│  SSH Connection Layer (RFC 4254)          │  canais multiplexados (shell, sftp, x11)
├──────────────────────────────────────────┤
│  SSH Authentication Layer (RFC 4252)      │  publickey, password, keyboard-interactive
├──────────────────────────────────────────┤
│  SSH Transport Layer (RFC 4253)           │  KEX, cifra simétrica, MAC, compressão
├──────────────────────────────────────────┤
│  TCP/IP                                   │  porta 22 por padrão
└──────────────────────────────────────────┘
```

### Handshake detalhado

```
Client                              Server
  |                                    |
  |──── TCP SYN ──────────────────────>|
  |<─── TCP SYN+ACK ───────────────────|
  |                                    |
  |──── SSH banner (SSH-2.0-...) ─────>|
  |<─── SSH banner ─────────────────── |
  |                                    |
  |──── KEXINIT (algoritmos) ─────────>|  client propõe: kex, cifras, MACs
  |<─── KEXINIT ───────────────────────|  server responde com suporte
  |                                    |
  |──── KEX (ex: curve25519-sha256) ──>|  troca de chaves Diffie-Hellman
  |<─── KEX reply + host key ──────────|  server envia chave pública do host
  |                                    |
  |  [derivação de session keys]        |  ambos derivam: enc_key, mac_key, IV
  |                                    |
  |──── NEWKEYS ───────────────────────>|  a partir daqui: tudo cifrado
  |<─── NEWKEYS ────────────────────── |
  |                                    |
  |──── AUTH request ─────────────────>|  "quero autenticar como user X"
  |  [publickey / password / etc]       |
  |<─── AUTH success ──────────────────|
  |                                    |
  |──── CHANNEL_OPEN (session) ────────>|  abre canal para shell/exec/sftp
  |<─── CHANNEL_OPEN_CONFIRM ──────────|
```

> [!info] Forward Secrecy
> O KEX com Diffie-Hellman garante **Perfect Forward Secrecy**: mesmo que a chave privada do servidor vaze no futuro, sessões passadas não podem ser decifradas porque as session keys efêmeras já foram descartadas.

### Autenticação por chave pública — mecanismo

```
1. Cliente envia: "quero autenticar com esta chave pública K"
2. Servidor verifica: K está em ~/.ssh/authorized_keys?
3. Servidor gera challenge: número aleatório cifrado com K (pública)
4. Cliente decifra com chave privada → prova que possui a privada
5. Servidor valida → autenticação bem-sucedida
```

A chave privada **nunca sai** da máquina cliente. Só a prova criptográfica trafega.

### Modelo de processos do sshd

```
sshd (listener) — PID master
  └── sshd (pre-auth) — por conexão entrante
        └── sshd (post-auth) — após autenticação, roda como o usuário
              └── bash (ou shell do usuário)
```

O processo post-auth herda os grupos e permissões do usuário. `sshd` usa **privilege separation** por padrão (`UsePrivilegeSeparation yes`): o processo não-privilegiado lida com dados da rede; operações privilegiadas (chaves host, PAM) ficam no processo monitor.

---

## Na prática — comandos e exemplos reais

### Conectar

```bash
# básico
ssh user@host

# porta diferente
ssh -p 2222 user@host

# chave específica
ssh -i ~/.ssh/id_ed25519 user@host

# forçar IPv4
ssh -4 user@host

# verbose (diagnóstico) — use -vvv para máximo
ssh -v user@host
```

### Geração de chaves

```bash
# ed25519 — recomendado (curva elíptica moderna, chave menor, mais rápida)
ssh-keygen -t ed25519 -C "felipe@homelab" -f ~/.ssh/id_ed25519

# RSA 4096 — para compatibilidade com sistemas legados
ssh-keygen -t rsa -b 4096 -C "felipe@homelab" -f ~/.ssh/id_rsa

# Mudar passphrase sem gerar nova chave
ssh-keygen -p -f ~/.ssh/id_ed25519
```

> [!warning] Nunca use RSA < 2048 ou DSA
> DSA tem tamanho de chave fixo em 1024 bits — quebrado. RSA < 2048 é considerado fraco. Use ed25519 por padrão.

### Copiar chave pública

```bash
# copia ~/.ssh/id_ed25519.pub para authorized_keys no servidor
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host

# manualmente:
cat ~/.ssh/id_ed25519.pub | ssh user@host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### SSH Agent — evitar digitar passphrase repetidamente

```bash
# iniciar agent na sessão atual
eval "$(ssh-agent -s)"

# adicionar chave (com passphrase, pede uma vez)
ssh-add ~/.ssh/id_ed25519

# listar chaves no agent
ssh-add -l

# remover todas as chaves
ssh-add -D

# agent com TTL (remove chave após 4 horas)
ssh-add -t 14400 ~/.ssh/id_ed25519
```

### Config do cliente (`~/.ssh/config`)

```sshconfig
# Host específico com alias
Host k8s-master
    HostName 192.168.3.30
    User felipe
    IdentityFile ~/.ssh/id_ed25519
    Port 22

# Todos os hosts da rede homelab
Host 192.168.3.*
    User felipe
    IdentityFile ~/.ssh/id_ed25519
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

# Configuração global de performance
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ControlMaster auto
    ControlPath ~/.ssh/cm_%r@%h:%p
    ControlPersist 10m
    AddKeysToAgent yes
```

> [!tip] ControlMaster / Multiplexing
> `ControlMaster auto` + `ControlPath` fazem com que múltiplas conexões para o mesmo host **reusem o mesmo TCP/TLS** já estabelecido. Segundo `ssh` para o mesmo host conecta em milissegundos. Crítico para scripts que abrem muitas conexões.

### Port Forwarding

```bash
# Local forwarding: acessa porta remota como se fosse local
# Acessa postgres em 192.168.3.30:5432 via localhost:5432
ssh -L 5432:localhost:5432 felipe@192.168.3.30

# Remote forwarding: expõe porta local no servidor remoto
# Servidor remoto pode acessar seu localhost:3000 via 0.0.0.0:8080
ssh -R 8080:localhost:3000 felipe@servidor-remoto

# Dynamic forwarding: proxy SOCKS5
ssh -D 1080 felipe@jumphost
# configure navegador para usar SOCKS5 localhost:1080

# Sem abrir shell (só o túnel)
ssh -N -L 5432:localhost:5432 felipe@192.168.3.30

# Background
ssh -fN -L 5432:localhost:5432 felipe@192.168.3.30
```

### Execução remota sem shell interativa

```bash
# rodar comando e retornar
ssh felipe@192.168.3.30 "sudo kubectl get nodes"

# pipe funciona normalmente
ssh felipe@192.168.3.30 "cat /var/log/syslog" | grep ERROR

# transferir diretório sem scp (tar via ssh)
tar czf - ./dir | ssh user@host "tar xzf - -C /destino"

# rodar script local no servidor remoto
ssh user@host bash < script-local.sh
```

### SCP e SFTP

```bash
# copiar arquivo para servidor
scp arquivo.txt felipe@192.168.3.30:/tmp/

# copiar diretório recursivo
scp -r ./configs felipe@192.168.3.30:/etc/myapp/

# copiar com chave específica
scp -i ~/.ssh/id_ed25519 arquivo.txt felipe@192.168.3.30:/tmp/

# sftp interativo
sftp felipe@192.168.3.30

# sftp batch (não-interativo)
sftp -b comandos.txt felipe@192.168.3.30
```

> [!info] SCP vs rsync
> `scp` é simples mas não faz delta transfer. Para sincronizar diretórios grandes use `rsync -avz --progress -e ssh src/ user@host:/dst/`

---

## Configuração do servidor (`/etc/ssh/sshd_config`)

```sshconfig
# Porta (mudar reduz ruído de logs, não é segurança real)
Port 22

# Protocolo — apenas SSH-2
Protocol 2

# Autenticação — hardening recomendado
PermitRootLogin no                  # nunca root direto
PasswordAuthentication no           # só chave pública
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
ChallengeResponseAuthentication no
UsePAM yes

# Limites de sessão
LoginGraceTime 30s                  # tempo máximo para autenticar
MaxAuthTries 3                      # tentativas antes de fechar conexão
MaxSessions 10                      # canais por conexão

# Restrições de acesso
AllowUsers felipe deploy            # whitelist de usuários
AllowGroups sudo developers         # ou por grupo
DenyUsers root                      # blacklist

# Tunelamento
AllowTcpForwarding yes
GatewayPorts no                     # remote forwarding só em loopback
X11Forwarding no

# Keepalive
ClientAliveInterval 60
ClientAliveCountMax 3

# Subsistemas
Subsystem sftp /usr/lib/openssh/sftp-server

# Restrições por bloco (Match)
Match User deploy
    ForceCommand /usr/local/bin/deploy.sh
    AllowTcpForwarding no
    X11Forwarding no

Match Address 192.168.3.0/24
    PasswordAuthentication yes      # permite senha só na rede local
```

Após editar:
```bash
# validar sintaxe antes de restartar
sudo sshd -t

# recarregar sem derrubar sessões existentes
sudo systemctl reload sshd
```

---

## Casos de uso e boas práticas

### Segurança — hardening checklist

- [ ] `PermitRootLogin no` — sempre
- [ ] `PasswordAuthentication no` — prefira chaves
- [ ] Chaves ed25519 ou RSA 4096
- [ ] Passphrase nas chaves privadas (use ssh-agent)
- [ ] `AllowUsers` ou `AllowGroups` — whitelist explícita
- [ ] `LoginGraceTime 30` e `MaxAuthTries 3`
- [ ] fail2ban ou similar para bloquear brute-force
- [ ] `ClientAliveInterval 60` para detectar conexões mortas
- [ ] Porta não-padrão (reduz log noise, não é segurança real)
- [ ] Auditoria: `AuthorizedKeysCommand` para centralizar chaves

### Centralização de chaves em produção

Em vez de gerenciar `authorized_keys` por máquina, use:
- **LDAP/AD + AuthorizedKeysCommand** — consulta chave do usuário em tempo real
- **HashiCorp Vault SSH CA** — assina chaves temporárias, sem `authorized_keys`
- **AWS SSM Session Manager / GCP IAP** — sem SSH direto, sem portas abertas

### Bastion / Jump host

```bash
# saltar pelo bastion sem configurar nada no intermediário
ssh -J bastion.prod user@servidor-interno

# no ~/.ssh/config
Host servidor-interno
    ProxyJump bastion.prod
    # ou ProxyCommand ssh -W %h:%p bastion.prod (versão antiga)

# múltiplos saltos
ssh -J bastion1,bastion2 user@destino-final
```

---

## Troubleshooting — cenários reais de produção

### "Permission denied (publickey)"

```bash
# 1. verbose no cliente para ver qual chave está sendo ofertada
ssh -vvv user@host 2>&1 | grep -E "(Offering|Authentications|denied)"

# 2. verificar permissões no servidor (causa mais comum)
ls -la ~/.ssh/
# ~/.ssh deve ser 700, authorized_keys deve ser 600
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 644 ~/.ssh/*.pub

# 3. verificar contexto SELinux/AppArmor
ls -Z ~/.ssh/authorized_keys
restorecon -v ~/.ssh/authorized_keys

# 4. log do servidor
sudo journalctl -u sshd -f
sudo tail -f /var/log/auth.log   # Debian/Ubuntu
```

### "Connection refused" ou timeout

```bash
# verificar se sshd está rodando e escutando
sudo systemctl status sshd
sudo ss -tlnp | grep :22

# verificar firewall
sudo iptables -L INPUT -n -v | grep 22
sudo ufw status

# testar conectividade básica
nc -zv host 22
telnet host 22
```

### "Host key verification failed"

```bash
# ocorre quando a chave do servidor mudou (reinstalação, VM recriada)
# verificar qual linha está conflitando
ssh-keygen -R hostname
ssh-keygen -R 192.168.3.30

# para ambientes de lab onde hosts mudam frequentemente:
# adicionar no ~/.ssh/config do host:
#   StrictHostKeyChecking no
#   UserKnownHostsFile /dev/null
```

> [!warning] TOFU (Trust On First Use)
> Por padrão, na primeira conexão o cliente aceita a chave do servidor sem validação — é o modelo TOFU. Em produção, distribua as known_hosts via configuração gerenciada (Ansible, cloud-init) para verificar autenticidade real do servidor.

### SSH lento para conectar

```bash
# causa 1: DNS reverso lento (sshd tenta resolver o IP do cliente)
# adicionar em sshd_config:
# UseDNS no

# causa 2: GSSAPI (Kerberos) tentando autenticar
# no cliente ~/.ssh/config:
# GSSAPIAuthentication no

# causa 3: keep-alive não configurado, TCP half-open
# ClientAliveInterval 60 no servidor
# ServerAliveInterval 60 no cliente

# diagnóstico de tempo
time ssh user@host exit
ssh -vvv user@host exit 2>&1 | grep -E "^debug1: (Authenticat|connect)"
```

### Sessão SSH cai durante operações longas

```bash
# causas: idle timeout, NAT timeout, firewall stateful
# solução no cliente:
ssh -o ServerAliveInterval=60 -o ServerAliveCountMax=3 user@host

# ou no ~/.ssh/config:
# ServerAliveInterval 60

# para nohup em sessões longas: use tmux/screen no servidor
ssh user@host
tmux new -s trabalho
# ctrl+b d para detach — sessão persiste mesmo com SSH caído
```

---

## Nível avançado — edge cases, flags menos conhecidas, Staff-level

### Algoritmos modernos — auditoria e hardening

```bash
# ver algoritmos negociados na conexão
ssh -vvv user@host 2>&1 | grep -E "(kex|cipher|mac|host key)"

# auditoria com ssh-audit (ferramenta especializada)
ssh-audit host:22

# hardening de algoritmos em sshd_config (eliminar algoritmos fracos)
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
HostKeyAlgorithms ssh-ed25519,rsa-sha2-512,rsa-sha2-256
```

### Certificados SSH (CA)

```bash
# criar CA
ssh-keygen -t ed25519 -f ssh_ca -C "homelab-ca"

# assinar chave de usuário (válida por 1 dia, para usuário felipe)
ssh-keygen -s ssh_ca -I "felipe@homelab" -n felipe -V +1d ~/.ssh/id_ed25519.pub

# resultado: id_ed25519-cert.pub — arquivo certificado
# o usuário usa normalmente: ssh -i ~/.ssh/id_ed25519 user@host

# no servidor: confiar na CA em vez de authorized_keys
# /etc/ssh/sshd_config:
# TrustedUserCAKeys /etc/ssh/ssh_ca.pub

# vantagem: revogar acesso centralmente sem tocar em cada servidor
# criar arquivo CRL:
ssh-keygen -k -f revoked_keys -s ssh_ca serial_numbers.txt
# RevokedKeys /etc/ssh/revoked_keys
```

> [!tip] CA SSH vs authorized_keys
> Em infraestruturas com muitos servidores, manter `authorized_keys` por máquina é operacionalmente inviável. CA SSH centraliza o trust: você confia na CA, e a CA assina certificados temporários. Quando um funcionário sai, você revoga o certificado — sem precisar tocar em centenas de servidores.

### ProxyJump e Agent Forwarding — diferença crítica

```bash
# Agent Forwarding (EVITAR em produção)
ssh -A user@bastion
# O agente do cliente fica disponível no bastion
# RISCO: root no bastion pode usar sua chave para autenticar em qualquer lugar

# ProxyJump (PREFERIR)
ssh -J bastion user@servidor-interno
# A conexão para servidor-interno é estabelecida pelo CLIENT, não pelo bastion
# O bastion apenas retransmite bytes TCP — nunca vê a chave privada
```

### Multiplexing avançado

```bash
# verificar canais ativos no master
ssh -O check -S ~/.ssh/cm_felipe@192.168.3.30:22 ""

# enviar sinal de saída ao master
ssh -O exit -S ~/.ssh/cm_felipe@192.168.3.30:22 ""

# abrir canal adicional sem novo handshake
ssh -S ~/.ssh/cm_felipe@192.168.3.30:22 felipe@192.168.3.30 "ls /tmp"
```

### Escape sequences — controle da sessão

Com conexão SSH aberta, pressione `~` (tilde) seguido de:

| Sequência | Ação |
|---|---|
| `~.` | Fechar conexão (útil quando shell travou) |
| `~^Z` | Suspender ssh para background |
| `~#` | Listar conexões encaminhadas |
| `~&` | Background e logout |
| `~~` | Enviar tilde literal |

### SSHFS — montar filesystem remoto

```bash
# instalar
sudo apt install sshfs

# montar
mkdir -p ~/mnt/k8s-master
sshfs felipe@192.168.3.30:/home/felipe ~/mnt/k8s-master

# montar com opções
sshfs -o reconnect,ServerAliveInterval=15,ServerAliveCountMax=3 \
    felipe@192.168.3.30:/ ~/mnt/k8s-master

# desmontar
fusermount -u ~/mnt/k8s-master
```

### Debugging profundo — strace no sshd

```bash
# quando você precisa entender o que o sshd está fazendo por dentro
sudo strace -f -p $(pgrep sshd | head -1) -e trace=network,file 2>&1 | grep -v EAGAIN

# rastrear autenticação específica
sudo strace -f -e trace=read,write,open,connect \
    /usr/sbin/sshd -d -p 2222 2>&1 | grep auth
```

### `authorized_keys` — opções avançadas por linha

```
# restrição de comando (deploy keys)
command="/usr/local/bin/deploy.sh",no-port-forwarding,no-X11-forwarding,no-agent-forwarding ssh-ed25519 AAAA... deploy@ci

# restrição por IP de origem
from="192.168.3.0/24",no-pty ssh-ed25519 AAAA... monitoramento

# túnel específico apenas
permitopen="localhost:5432",no-pty,command="/bin/false" ssh-ed25519 AAAA... db-tunnel

# chave temporária com expiração (requer CA)
# use certificados SSH com -V para isso
```

### Performance — benchmarking de cifras

```bash
# comparar throughput de cifras
openssl speed aes-256-cbc aes-256-gcm chacha20

# medir throughput de transferência SSH
dd if=/dev/zero bs=1M count=1000 | ssh user@host "cat > /dev/null"

# usar cifra mais rápida para transferências internas (ambiente confiável)
scp -o "Ciphers=aes128-gcm@openssh.com" arquivo.tar user@host:/tmp/
```

---

## Referências

- [OpenSSH Manual Pages](https://www.openssh.com/manual.html)
- RFC 4251 — SSH Protocol Architecture
- RFC 4252 — SSH Authentication Protocol
- RFC 4253 — SSH Transport Layer Protocol
- RFC 4254 — SSH Connection Protocol
- [ssh-audit](https://github.com/jtesta/ssh-audit) — ferramenta de auditoria de configuração SSH
- [Mozilla SSH Guidelines](https://infosec.mozilla.org/guidelines/openssh) — configurações recomendadas por nível de segurança
- `man sshd_config` / `man ssh_config` / `man ssh-keygen`
