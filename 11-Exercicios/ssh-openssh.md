---
tags:
  - exercicios
  - networking
  - linux
  - ssh
  - openssh
tipo: exercicios
area: networking
conteudo: "[[02-Networking/ssh-openssh]]"
trilha: "[[00-Trilha/networking]]"
---

# Exercícios — SSH / OpenSSH

> [!info] Ambiente
> Homelab: k8s-master (`192.168.3.30`), k8s-worker1 (`192.168.3.31`), k8s-worker2 (`192.168.3.32`)
> Execute os exercícios da sua máquina local ou de um dos nodes.
> **Não altere namespaces de produção** (`controle-gastos`, `promobot`).

---

## Exercício 1: Inventário de chaves e autenticação atual

**Contexto:** Você acabou de assumir um servidor. Antes de qualquer mudança, precisa entender o estado atual da autenticação SSH.

**Missão:** Mapear chaves existentes, métodos de autenticação habilitados e usuários com acesso SSH no `k8s-master`.

**Requisitos:**
- [ ] Listar todos os arquivos em `~/.ssh/` com permissões
- [ ] Identificar que tipo de chave está sendo usada atualmente (algoritmo)
- [ ] Verificar quais métodos de autenticação estão habilitados em `/etc/ssh/sshd_config`
- [ ] Verificar quais usuários têm `authorized_keys` configurado (checar home directories)
- [ ] Identificar a versão do OpenSSH no servidor

**Verificação:**
```bash
ssh -v user@192.168.3.30 exit 2>&1 | grep -E "(Server version|Authentications|key type)"
```

---

## Exercício 2: Geração e deploy de chave ed25519

**Contexto:** A empresa padronizou ed25519 como algoritmo de chave SSH. Você precisa criar uma nova chave, com passphrase, e configurar acesso ao k8s-master sem usar a chave antiga.

**Missão:** Criar par de chaves ed25519 e configurar acesso ao k8s-master usando apenas ela.

**Requisitos:**
- [ ] Gerar chave ed25519 com comentário identificando máquina e propósito
- [ ] Proteger a chave privada com passphrase
- [ ] Copiar a chave pública para o k8s-master sem usar `ssh-copy-id` (manualmente)
- [ ] Conectar usando a nova chave especificando com `-i`
- [ ] Verificar as permissões corretas do `~/.ssh/` e `authorized_keys` no servidor

**Verificação:**
```bash
ssh -i ~/.ssh/NOME_DA_SUA_CHAVE felipe@192.168.3.30 "echo autenticado"
```

---

## Exercício 3: SSH Config e aliases

**Contexto:** Você acessa os 3 nodes do homelab diariamente. Digitar IP e usuário toda vez é ineficiente. Além disso, precisa de um alias `homelab-jump` para conectar ao worker2 passando pelo master.

**Missão:** Configurar `~/.ssh/config` com aliases para os 3 nodes e um acesso via ProxyJump.

**Requisitos:**
- [ ] Criar entradas `k8s-master`, `k8s-worker1`, `k8s-worker2` no config
- [ ] Configurar a chave ed25519 criada no exercício anterior para todos os hosts
- [ ] Criar entrada `k8s-worker2-jump` que acessa o worker2 via ProxyJump pelo master
- [ ] Conectar usando `ssh k8s-master` (sem IP, usuário ou `-i`)
- [ ] Conectar usando `ssh k8s-worker2-jump`
- [ ] Habilitar ControlMaster com persistência de 5 minutos para todos os hosts

**Verificação:**
```bash
time ssh k8s-master exit
time ssh k8s-master exit  # segunda conexão deve ser significativamente mais rápida
```

---

## Exercício 4: Hardening do sshd_config

**Contexto:** Auditoria de segurança apontou que o servidor aceita senha como método de autenticação e não tem limites de tentativas de login. Você precisa fazer hardening sem perder seu próprio acesso.

**Missão:** Aplicar configurações de segurança no `sshd_config` do k8s-master.

**Requisitos:**
- [ ] Desabilitar autenticação por senha (`PasswordAuthentication no`)
- [ ] Desabilitar login direto como root
- [ ] Configurar `MaxAuthTries 3` e `LoginGraceTime 30`
- [ ] Validar o sshd_config com `sshd -t` **antes** de recarregar
- [ ] Recarregar o sshd sem matar sessões existentes (não `restart`, use `reload`)
- [ ] Confirmar que autenticação por senha foi bloqueada tentando conectar com `ssh -o PubkeyAuthentication=no`
- [ ] Garantir que sua chave pública ainda funciona após o hardening

**Verificação:**
```bash
ssh -o PubkeyAuthentication=no -o PasswordAuthentication=yes felipe@192.168.3.30
# deve recusar — "Permission denied (publickey)"
ssh k8s-master "echo acesso ok"
# deve funcionar
```

---

## Exercício 5: SSH Agent e forwarding seguro

**Contexto:** Você precisa, a partir do k8s-master, conectar no k8s-worker1 sem copiar sua chave privada para o master. O time de segurança proibiu `ForwardAgent` por risco de comprometimento.

**Missão:** Conectar do master ao worker1 usando ProxyJump (sem ForwardAgent, sem copiar chave privada).

**Requisitos:**
- [ ] Confirmar que sua chave privada **não existe** em `~/.ssh/` no k8s-master
- [ ] Criar entrada no `~/.ssh/config` local para acessar k8s-worker1 via ProxyJump pelo master
- [ ] Conectar no k8s-worker1 a partir da **sua máquina local**, sem parar pelo master interativamente
- [ ] Executar `hostname` remotamente no worker1 para confirmar onde está
- [ ] Explicar a diferença técnica entre ProxyJump e ForwardAgent

**Verificação:**
```bash
ssh k8s-worker1-via-master hostname
# deve retornar: k8s-worker1 (ou similar)
```

---

## Exercício 6: Port forwarding — acesso a serviços internos

**Contexto:** O PostgreSQL roda no homelab na rede interna. Você precisa acessá-lo da sua máquina local sem expor a porta 5432 diretamente na rede.

**Missão:** Criar túnel SSH para acessar o PostgreSQL via k8s-master, em background, sem shell interativa.

**Requisitos:**
- [ ] Identificar onde o PostgreSQL está rodando (IP e porta) no homelab
- [ ] Criar local port forward mapeando `localhost:15432` → `postgres-host:5432` via k8s-master
- [ ] O processo SSH deve rodar em background (não ocupar o terminal)
- [ ] Verificar que o túnel está ativo com `ss` ou `netstat`
- [ ] Conectar no PostgreSQL usando `psql -h localhost -p 15432`
- [ ] Matar o processo de túnel quando terminar

**Verificação:**
```bash
ss -tlnp | grep 15432
psql -h localhost -p 15432 -U postgres -c "SELECT version();"
```

---

## Exercício 7: Diagnóstico — "não consigo conectar"

**Contexto:** Um colega reporta: *"não consigo mais fazer SSH no k8s-worker2, era pra funcionar com minha chave"*. Você precisa diagnosticar sem acesso inicial ao servidor (use o master como ponto de entrada).

**Missão:** Diagnosticar sistematicamente um problema de autenticação SSH seguindo um runbook mental.

**Requisitos:**
- [ ] Checar conectividade TCP na porta 22 do worker2 a partir do master
- [ ] Verificar se sshd está rodando no worker2 (via master)
- [ ] Checar permissões do `~/.ssh/` e `authorized_keys` do colega no worker2
- [ ] Verificar logs do sshd no momento da tentativa de login
- [ ] Documentar o que encontrou e qual seria a correção

**Verificação:**
```bash
# a partir do master, simular o problema:
ssh -vvv -i /tmp/chave-colega usuario@192.168.3.32 2>&1 | tail -20
```

---

## Exercício 8: SSH CA — infraestrutura de certificados (Staff-level)

**Contexto:** Com 3 nodes no homelab, manter `authorized_keys` em cada um está ficando tedioso. Você quer implementar uma CA SSH onde os servidores confiam na CA e os usuários usam certificados assinados temporários.

**Missão:** Criar uma mini-infraestrutura de CA SSH no homelab.

**Requisitos:**
- [ ] Criar par de chaves CA em `/etc/ssh/ssh_ca` no k8s-master (ou local)
- [ ] Configurar o k8s-worker1 para confiar na CA (`TrustedUserCAKeys`)
- [ ] Assinar sua chave pública ed25519 com validade de 2 horas, principal `felipe`
- [ ] Conectar no worker1 usando o certificado (sem `authorized_keys`)
- [ ] Verificar o certificado com `ssh-keygen -L -f id_ed25519-cert.pub`
- [ ] Aguardar expiração (ou ajustar o relógio) e confirmar que o certificado expirado é rejeitado
- [ ] Explicar como isso escala para 100 servidores vs `authorized_keys`

**Verificação:**
```bash
ssh-keygen -L -f ~/.ssh/id_ed25519-cert.pub
ssh -i ~/.ssh/id_ed25519 -o CertificateFile=~/.ssh/id_ed25519-cert.pub felipe@192.168.3.31
```
