---
tags:
  - kubernetes
  - secrets
  - segurança
  - credenciais
area: kubernetes
tipo: conteudo
prerequisites:
  - "[[05-Kubernetes/configmaps]]"
  - "[[05-Kubernetes/volumes-storage]]"
next:
  - "[[11-Exercicios/secrets]]"
trilha: "[[00-Trilha/kubernetes]]"
---

# Secrets no Kubernetes

> Conteúdo: [[05-Kubernetes/secrets]] | Exercícios: [[11-Exercicios/secrets]] | Trilha: [[00-Trilha/kubernetes]]

Secrets são o mecanismo do Kubernetes para armazenar dados sensíveis: senhas, tokens, chaves TLS, credenciais de registry. A API é similar à de [[05-Kubernetes/configmaps]], mas com garantias adicionais de isolamento e acesso controlado. Entender Secrets é entender também seus limites — base64 não é criptografia, e o modelo de segurança depende muito de configuração complementar.

---

## O que é e por que existe

### ConfigMap vs Secret — a distinção real

Tecnicamente, um Secret é um ConfigMap com dois comportamentos adicionais:

1. **Armazenamento separado no etcd** — Secrets podem ser criptografados em repouso no etcd (`EncryptionConfiguration`), ConfigMaps não
2. **Montagem em tmpfs** — quando montado como volume, o kubelet usa um filesystem em memória (tmpfs) em vez de disco, evitando que dados sensíveis sejam escritos em disco no node
3. **RBAC granular** — é possível dar permissão de leitura a ConfigMaps sem dar para Secrets, e vice-versa

### Por que não colocar senhas em ConfigMaps

- ConfigMaps aparecem em texto puro no `kubectl get cm -o yaml`
- ConfigMaps não são criptografados no etcd por padrão
- Logs de audit, dumps de etcd, backups — todos expõem ConfigMaps sem nenhuma proteção adicional

> [!warning] base64 ≠ criptografia
> O conteúdo de um Secret é armazenado em **base64** — não criptografado. `echo "cGFzc3dvcmQ=" | base64 -d` devolve `password`. Qualquer pessoa com acesso ao etcd ou permissão de `kubectl get secret` lê os valores. A proteção real vem de RBAC + EncryptionConfiguration + práticas de GitOps seguras.

---

## Como funciona internamente

### Tipos de Secret

| Tipo | Campo `type` | Uso |
|---|---|---|
| **Opaque** | `Opaque` | Dados arbitrários — o tipo padrão |
| **TLS** | `kubernetes.io/tls` | Certificado (`tls.crt`) e chave (`tls.key`) |
| **Docker Registry** | `kubernetes.io/dockerconfigjson` | Credenciais de registry privado |
| **Service Account Token** | `kubernetes.io/service-account-token` | Token para ServiceAccount (gerado automaticamente) |
| **Basic Auth** | `kubernetes.io/basic-auth` | `username` e `password` |
| **SSH Auth** | `kubernetes.io/ssh-auth` | Chave SSH (`ssh-privatekey`) |
| **Bootstrap Token** | `bootstrap.kubernetes.io/token` | Tokens de bootstrap de nodes |

### Como o kubelet entrega Secrets aos Pods

**Como variável de ambiente:**
O valor é lido do etcd no momento do start do Pod e injetado como variável no processo. O valor fica em `/proc/<PID>/environ` — visível para qualquer processo com acesso ao `/proc` do host.

**Como volume (método preferido):**
O kubelet monta um `tmpfs` (RAM) no path do volume. Cada chave do Secret vira um arquivo. O tmpfs não é persistido em disco. Quando o Pod termina, o tmpfs é desmontado e os dados somem da memória.

```
/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~secret/<secret-name>/
  └── password     ← arquivo com o valor (sem base64)
  └── username     ← arquivo com o valor
```

### Secret watch e atualização automática

Quando um Secret é atualizado, o kubelet detecta a mudança (via watch na API) e atualiza os arquivos no volume em até **~60 segundos** (kubelet sync period). Variáveis de ambiente **não são atualizadas** — o processo não recebe as novas variáveis até ser reiniciado.

```
Volume Secret  →  atualização automática em ~60s (aplicação precisa recarregar o arquivo)
Env var Secret →  NUNCA atualiza — requer restart do Pod
```

### Secrets no etcd — o risco real

Por padrão, o etcd armazena Secrets em base64 sem criptografia adicional. Para habilitar criptografia em repouso:

```yaml
# /etc/kubernetes/encryption-config.yaml
kind: EncryptionConfiguration
apiVersion: apiserver.config.k8s.io/v1
resources:
- resources:
  - secrets
  providers:
  - aescbc:        # ou aesgcm, secretbox
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}   # fallback para Secrets não criptografados
```

> [!info] Homelab — etcd encryption
> No seu cluster, verifique se a EncryptionConfiguration está ativa:
> `sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep encryption`
> Se não estiver configurada, Secrets estão em base64 puro no etcd.

---

## Na prática — comandos e YAMLs

### Criação imperativa

```bash
# --from-literal: par chave=valor direto
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password='s3cr3t!'

# --from-file: arquivo inteiro vira o valor da chave
kubectl create secret generic tls-cert \
  --from-file=tls.crt=/path/to/cert.pem \
  --from-file=tls.key=/path/to/key.pem

# --from-env-file: arquivo .env (chave=valor por linha)
kubectl create secret generic app-env \
  --from-env-file=.env

# dry-run para gerar YAML (e revisar antes de aplicar)
kubectl create secret generic db-credentials \
  --from-literal=password='s3cr3t!' \
  --dry-run=client -o yaml
```

### YAML — Secret Opaque

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
type: Opaque
data:
  username: YWRtaW4=          # base64 de "admin"
  password: czNjcjN0IQ==      # base64 de "s3cr3t!"
```

```bash
# Gerar base64 corretamente (sem newline)
echo -n "s3cr3t!" | base64       # -n omite o \n final
# czNjcjN0IQ==

# Decodificar
echo "czNjcjN0IQ==" | base64 -d
# s3cr3t!
```

> [!warning] Sempre use `echo -n` para gerar base64
> `echo "senha" | base64` gera base64 de "senha\n" — o newline faz parte do valor. Isso causa erros silenciosos onde a senha tem um caractere extra. Use sempre `echo -n`.

### `stringData` — escrever em texto puro (Kubernetes converte automaticamente)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:         # aceita texto puro — Kubernetes faz o base64
  username: admin
  password: "s3cr3t!"
  config.yaml: |    # arquivos multi-line também funcionam
    host: postgres
    port: 5432
    database: mydb
```

> [!tip] Prefira stringData no YAML
> `stringData` é mais legível e menos sujeito a erros que base64 manual. O Kubernetes converte para `data` (base64) ao armazenar. O `kubectl get secret -o yaml` sempre exibe `data` (base64), nunca `stringData`.

### Consumindo como variável de ambiente

```yaml
# Chave específica
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password
      optional: false    # false = Pod falha se o Secret não existir

# Todas as chaves do Secret como variáveis
envFrom:
- secretRef:
    name: db-credentials
  prefix: DB_     # opcional: prefixa todas as variáveis com DB_
```

### Consumindo como volume (método seguro)

```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: db-secret
      mountPath: /run/secrets/db    # padrão de path para secrets
      readOnly: true
  volumes:
  - name: db-secret
    secret:
      secretName: db-credentials
      defaultMode: 0400             # apenas o dono pode ler (root no container)
      items:                        # opcional: montar apenas algumas chaves
      - key: password
        path: db-password           # nome do arquivo dentro do mountPath
```

```bash
# Dentro do container, ler o secret:
cat /run/secrets/db/db-password
# s3cr3t!

# Verificar que é tmpfs (não em disco)
df -h /run/secrets/db
# tmpfs ... tmpfs
```

### Secret TLS — para Ingress e cert-manager

```yaml
# Secret TLS criado manualmente
apiVersion: v1
kind: Secret
metadata:
  name: webapp-tls
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: <base64-do-certificado>
  tls.key: <base64-da-chave-privada>
```

```bash
# Criar Secret TLS a partir de arquivos
kubectl create secret tls webapp-tls \
  --cert=cert.pem \
  --key=key.pem

# O cert-manager cria Secrets TLS automaticamente
# Exemplo de Secret TLS criado pelo cert-manager no homelab:
kubectl get secret -n default | grep tls
kubectl describe secret <nome>-tls | grep -E "Type|Data"
```

### imagePullSecret — credenciais de registry privado

```bash
# Criar Secret para registry privado
kubectl create secret docker-registry registry-creds \
  --docker-server=registry.staypuff.info \
  --docker-username=felipe \
  --docker-password='token123' \
  --docker-email=felipe.staypuff@gmail.com
```

```yaml
# Usar no Pod
spec:
  imagePullSecrets:
  - name: registry-creds
  containers:
  - name: app
    image: registry.staypuff.info/myapp:latest
```

### Ler e decodificar Secrets

```bash
# Ver Secret (base64)
kubectl get secret db-credentials -o yaml

# Decodificar uma chave específica
kubectl get secret db-credentials \
  -o jsonpath='{.data.password}' | base64 -d

# Ver todas as chaves decodificadas (cuidado com logs)
kubectl get secret db-credentials -o json | \
  jq -r '.data | to_entries[] | "\(.key): \(.value | @base64d)"'

# Listar apenas os nomes das chaves (sem expor valores)
kubectl get secret db-credentials -o jsonpath='{.data}' | jq 'keys'
```

---

## Casos de uso e boas práticas

### Nunca commitar Secrets em Git

O fluxo errado (comum em projetos jovens):
```
secret.yaml com base64 → git commit → GitHub público → credencial exposta
```

**Soluções:**

1. **Secrets externalizados**: Sealed Secrets, External Secrets Operator, Vault Agent
2. **`.gitignore` para arquivos de secret**: adicionar `*-secret.yaml`, `.env`, `*.key`
3. **SOPS**: criptografa o YAML antes de commitar, decripta no deploy

```bash
# .gitignore
*-secret.yaml
*.key
*.pem
.env
```

### Volume vs variável de ambiente — quando usar cada um

| | Volume | Env Var |
|---|---|---|
| **Segurança** | Melhor (tmpfs, não em `/proc/environ`) | Pior (exposto em `/proc/<pid>/environ`) |
| **Atualização** | Automática (~60s) | Só com restart |
| **Multi-linha** | Funciona bem (arquivo) | Problemático |
| **Debugging** | Requer `exec` no container | `kubectl exec -- env` |
| **Compatibilidade** | Requer adaptação da app | Qualquer app lê env vars |

**Regra prática:** credenciais de banco, chaves TLS, tokens → volume. Configurações simples de auth → env var (aceitável se o risco for baixo).

### Imutable Secrets (K8s 1.21+)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
immutable: true    # não pode ser alterado após criação
```

Vantagens: protege contra mudanças acidentais e melhora performance (kubelet não precisa fazer watch).

### RBAC mínimo para Secrets

```yaml
# Role que só pode ler um Secret específico
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: read-db-secret
  namespace: default
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-credentials"]   # só este Secret específico
  verbs: ["get"]
```

> [!warning] Nunca dar `list` em Secrets sem necessidade
> `list secrets` retorna todos os Secrets do namespace incluindo os valores em base64. Dar `list` é quase equivalente a dar leitura de todos os valores. Use `resourceNames` para limitar ao mínimo necessário.

---

## Troubleshooting — cenários reais de produção

### Cenário 1: Pod em `CreateContainerConfigError`

```bash
kubectl describe pod <nome> | grep -A5 Events
# Warning  Failed  Error: secret "db-credentials" not found

# O Secret não existe no namespace correto
kubectl get secret db-credentials -n <namespace>
# Error from server (NotFound)

# Secrets são namespace-scoped — verificar namespace do Pod e do Secret
kubectl get pod <nome> -o jsonpath='{.metadata.namespace}'
kubectl get secret -n <namespace-correto>
```

### Cenário 2: Variável de ambiente com valor errado (newline extra)

```bash
# App falha na autenticação mas a senha parece correta

# Verificar o valor exato (incluindo newline)
kubectl exec <pod> -- bash -c 'echo -n "$DB_PASSWORD" | xxd | head'
# 00000000: 7333 6372 3374 210a  s3cr3t!.
#                           ^^ 0a = newline!

# Causa: echo sem -n ao criar o Secret
echo "s3cr3t!" | base64   # inclui \n
echo -n "s3cr3t!" | base64  # correto

# Corrigir o Secret
kubectl create secret generic db-credentials \
  --from-literal=password='s3cr3t!' \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Cenário 3: Secret atualizado mas app ainda usa valor antigo

```bash
# Situação: rotacionou a senha, atualizou o Secret, mas a app ainda usa a antiga

# 1. Verificar se o Secret foi atualizado
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d

# 2. Se for volume: verificar se o arquivo foi atualizado no Pod
kubectl exec <pod> -- cat /run/secrets/db/password
# (pode levar até 60s para propagar)

# 3. Se for env var: o Pod precisa reiniciar
kubectl rollout restart deployment/<nome>

# 4. Confirmar no Pod após restart
kubectl exec <novo-pod> -- env | grep DB_PASSWORD
```

### Cenário 4: Vazamento — Secret exposto em log ou ConfigMap

```bash
# Alguém acidentalmente logou o valor de uma variável de ambiente
kubectl logs <pod> | grep -i password

# Ação imediata:
# 1. Rotacionar a credencial no sistema externo (banco, API, etc.)
# 2. Atualizar o Secret com a nova credencial
kubectl create secret generic db-credentials \
  --from-literal=password='nova-senha' \
  --dry-run=client -o yaml | kubectl apply -f -
# 3. Reiniciar os Pods para pegar o novo valor
kubectl rollout restart deployment/<nome>
# 4. Invalidar a credencial antiga no sistema externo
```

### Cenário 5: imagePullBackOff por credenciais de registry

```bash
kubectl describe pod <nome> | grep -A5 Events
# Warning  Failed  Failed to pull image: unauthorized

# 1. Verificar se o imagePullSecret existe
kubectl get secret registry-creds
# Error (NotFound) → Secret não existe no namespace

# Secrets são namespace-scoped — se o Pod está em outro namespace
kubectl get secret registry-creds -n <namespace-do-pod>

# 2. Verificar se o Pod referencia o Secret correto
kubectl get pod <nome> -o jsonpath='{.spec.imagePullSecrets}'

# 3. Validar o conteúdo do Secret
kubectl get secret registry-creds \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .
```

---

## Nível avançado — edge cases e CKA/Staff

### Por que env vars são menos seguras que volumes

Variáveis de ambiente de um processo são legíveis em `/proc/<PID>/environ` — sem permissão especial para o processo em si:

```bash
# Dentro do container:
cat /proc/1/environ | tr '\0' '\n' | grep DB_PASSWORD
# DB_PASSWORD=s3cr3t!
```

Qualquer processo dentro do container pode ler as env vars de qualquer outro processo do mesmo namespace de PID. Com volume/tmpfs + `defaultMode: 0400`, apenas o processo que tem UID correto consegue ler o arquivo.

### Projeção de ServiceAccount Token (Bound Service Account Token)

O Kubernetes 1.21+ usa tokens de ServiceAccount com expiração e audiência via `projected` volume:

```yaml
volumes:
- name: sa-token
  projected:
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600      # expira em 1 hora
        audience: "https://kubernetes.default.svc"
```

O kubelet renova o token automaticamente antes de expirar. Tokens projetados são mais seguros que os antigos (sem expiração) e são o padrão desde K8s 1.24.

### External Secrets Operator — integração com Vault, AWS Secrets Manager

O External Secrets Operator cria Secrets Kubernetes automaticamente a partir de fontes externas:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-credentials   # nome do Secret K8s a criar
  data:
  - secretKey: password
    remoteRef:
      key: secret/prod/postgres
      property: password
```

### Sealed Secrets — commitar Secrets criptografados no Git

```bash
# Instalar kubeseal CLI
# Criar um Secret normal
kubectl create secret generic db-creds \
  --from-literal=password='s3cr3t!' \
  --dry-run=client -o yaml > secret.yaml

# Selar (criptografar com a chave pública do cluster)
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# sealed-secret.yaml pode ser commitado no Git — não tem como descriptografar sem a chave do cluster
kubectl apply -f sealed-secret.yaml
# O Sealed Secrets Controller decripta e cria o Secret real
```

### Auditando acesso a Secrets — audit log

```bash
# Ver quem acessou um Secret específico no audit log
grep '"resource":"secrets"' /var/log/kubernetes/audit.log | \
  jq '. | select(.objectRef.name == "db-credentials") | {user: .user.username, verb: .verb, time: .requestReceivedTimestamp}'
```

### cert-manager e Secrets TLS no homelab

O cert-manager usa Secrets do tipo `kubernetes.io/tls` para armazenar certificados. No seu cluster:

```bash
# Ver Secrets TLS criados pelo cert-manager
kubectl get secret -A | grep tls
kubectl describe secret <nome>-tls -n <namespace>
# Type: kubernetes.io/tls
# Data:
#   tls.crt: 4523 bytes
#   tls.key: 1679 bytes

# O Ingress referencia esses Secrets:
kubectl describe ingress <nome> | grep -A3 TLS
# TLS: <nome>-tls terminates <host>
```

---

## Referências

- **Kubernetes docs** — [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- **Kubernetes docs** — [Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- **Sealed Secrets** — github.com/bitnami-labs/sealed-secrets
- **External Secrets Operator** — external-secrets.io
- **SOPS** — github.com/mozilla/sops
- **Linuxtips DK8S** — `/mnt/c/Users/felip/Documents/Kubernets/DK8S`
- `kubectl explain secret.data` / `kubectl explain secret.stringData`
